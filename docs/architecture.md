# System Architecture

Status: baseline approved for implementation planning. This document records
the boundaries that code must preserve. A material change requires a numbered
ADR and an update to `PLAN.md`.

## Goals and invariants

The service accepts browser audio recordings from short clips through
multi-hour, 1GB+ files and returns a searchable, exportable transcript. The
following invariants are stronger than implementation convenience:

1. Production has zero VPS and no persistent server outside Cloudflare.
2. Large audio bytes do not cross a Worker control request. The browser and
   processing runtime use scoped, short-lived R2 access directly.
3. R2 is private and durable. D1 contains metadata/state only; Queue payloads
   contain IDs and bounded metadata only.
4. Container local disk is ephemeral scratch (`/tmp`). Restart or eviction
   cannot be the only place a source, chunk, transcript, or state exists.
5. Every mutating operation is authenticated, owner-scoped, bounded, and
   idempotent where retries are possible.
6. A successful job has no permanent audio: results are persisted first,
   source and successful temporary chunks are then removed promptly.
7. Standard transcription uses exactly `@cf/openai/whisper-large-v3-turbo`.
   Speaker diarization is optional and cannot introduce a non-Cloudflare
   production dependency.

## Runtime topology

```text
                         +----------------------+
                         | Browser (SPA)         |
                         | upload + transcript   |
                         +----------+-----------+
                                    |
                 control JSON       | direct S3/R2 data path
                                    |
                         +----------v-----------+       +---------------+
                         | Cloudflare Worker    |------>| private R2     |
                         | API + Static Assets  |       | original/temp  |
                         +---+-----------+------+       | results        |
                             |           |              +-------+-------+
                             |           |                      |
                            D1       Workflow                  |
                         metadata       |                      |
                                         v                      |
                              +----------+-----------+          |
                              | analyze/plan/merge   |<---------+
                              | durable job steps    |
                              +----------+-----------+
                                         |
                                  IDs only|
                                         v
                              +----------+-----------+
                              | Queue + chunk worker |
                              +----------+-----------+
                                         |
                                         v
                              +----------+-----------+
                              | Container            |
                              | ffprobe/FFmpeg       |
                              | ephemeral /tmp       |
                              +----------+-----------+
                                         |
                                         v
                              +----------+-----------+
                              | Workers AI adapter   |
                              | Whisper Turbo        |
                              +----------------------+
```

The Worker handles control-plane requests only: identity, job creation,
upload authorization, completion confirmation, status, result authorization,
speaker rename, and deletion. It can use an R2 binding for metadata and small
server-side artifacts, but it must reject or avoid proxying a large upload
body. Browser CORS is configured only for the application origin.

## Identity and ownership

The domain receives an `Identity` interface rather than reading cookies or
headers directly. The local/V1 baseline creates an opaque, high-entropy
HttpOnly, Secure, SameSite cookie and maps a hash of it to a `users` row. A
future account/federation adapter may replace this without changing job
ownership contracts. There is no public job identifier that grants access:
all job, upload, status, transcript, export, speaker, and delete operations
must compare the authenticated owner in D1.

CSRF protection is required for cookie-authenticated mutations. Session values,
R2 signatures, and transcript text are never logged. IDs use Web Crypto
randomness and are unguessable; object keys do not contain user-provided
filenames except as sanitized, non-authoritative metadata.

## Object and identifier layout

```text
original/{job_id}/source.{validated_extension}
temp/{job_id}/{chunk_id}.{format}
results/{job_id}/transcript.txt
results/{job_id}/transcript.md
results/{job_id}/transcript.json
results/{job_id}/transcript.srt
results/{job_id}/transcript.vtt
```

`job_id`, `chunk_id`, and `attempt_id` are separate identifiers. The original
source has exactly one completed object per job; an upload ID and incomplete
parts are not completed objects. Temporary chunk keys include a content or
plan version so stale retries cannot overwrite a newer plan. Result objects
include a schema/version marker and expiry metadata.

## Browser upload protocol

1. `POST /api/jobs` validates declared extension, MIME, byte size, mode, and
   an idempotency key; it creates `CREATED`/`UPLOADING` metadata.
2. `POST /api/jobs/{id}/upload-session` selects single PUT or multipart based
   on configurable size and policy. The Worker mints short-lived R2 temporary
   credentials (or equivalent per-operation signatures) scoped to one bucket
   and this job's `original/{id}/` path. A parent R2 token never reaches the
   browser.
3. The browser uses the S3-compatible endpoint directly. Multipart state
   (upload ID, part numbers, ETags, byte ranges, and expiry) is stored in the
   client and mirrored as bounded metadata in D1. Non-final parts are at least
   5 MiB; part size is calculated to remain under the 10,000-part limit.
4. Failed parts retry independently with bounded exponential backoff. A page
   reload resumes known parts; an expired session requests a newly scoped
   session without changing the job key. Abort is explicit and idempotent.
5. `POST /api/jobs/{id}/upload-complete` contains metadata only. The Worker
   checks owner, upload state, object key, R2 HEAD size/content type/etag, and
   expected checksum where available, then transitions to `UPLOADED` and
   starts one Workflow instance. It never accepts an arbitrary object key.

Presigned URLs are suitable for a single operation but do not provide a
multi-operation session; temporary credentials are preferred for browser
multipart. A capability test must verify the exact current S3 client/CORS
behavior before the DEV gate.

## Media analysis and chunk planning

The processing adapter receives a job and a short-lived source reference,
streams the source into a Container, and invokes ffprobe before any model
call. Required metadata is duration, codec, sample rate, channels, bitrate,
container format, and byte size. Extension and MIME are hints; ffprobe is the
authoritative format check. Commands use argument arrays and bounded resource
limits to avoid injection and denial-of-service paths.

FFmpeg emits an efficient encoded intermediate (preferably FLAC or another
verified model input) through a pipe or bounded `/tmp` file. It does not build
a whole-recording PCM WAV. Each chunk is uploaded to `temp/{job_id}/` before
the source stream advances, and local files are removed in `finally` blocks.
The source and chunk remain recoverable in R2 while a retry is possible, not on
Container disk.

The planner consumes a versioned `ModelCapabilities` record:

```text
inputEncoding, maxInputBytes, maxSafeDurationSeconds,
maxChunkCount, overlapSeconds, concurrency, timeoutSeconds,
capabilitySource, verifiedAt
```

Unknown required limits fail closed with an actionable error. Duration is a
starting heuristic, not a fixed contract: the planner considers encoded byte
estimates, codec/sample rate, speech density, model limits, and retry cost. It
produces deterministic chunk IDs and intervals that cover the source, with
5-15 seconds of overlap (clamped at file boundaries). Replanning increments
the plan version and leaves stale work unable to commit.

## AI provider boundary

```ts
interface TranscriptionProvider {
  transcribe(input: {
    chunkRef: string;
    offsetSeconds: number;
    languageHint?: string;
    mode: "standard" | "speaker";
  }): Promise<ProviderTranscript>;
}
```

The Workers AI adapter fetches a bounded chunk and calls the fixed Whisper
Turbo model. It records provider/model/version diagnostics and normalizes
`transcription_info`, `segments`, and available VTT/timestamp data into the
internal contract. It must classify retryable throttles/timeouts separately
from permanent invalid-input failures and must preflight byte/duration limits.

The current official model page documents batch audio input and structured
transcription segments, but it does not establish a diarization contract. A
speaker provider therefore returns optional local labels only; reconciliation
maps them to canonical speakers using overlap evidence and preserves
uncertainty. Speaker mode remains visibly unavailable until a current
Cloudflare-hosted model is verified for Chinese and mixed-language audio.

## Durable orchestration

One Workflow instance is keyed by `job_id` and contains only IDs, bounded
configuration, and references. Suggested idempotent steps:

```text
validate-upload -> analyze -> plan -> enqueue
    -> wait for all chunk outcomes
    -> merge -> export -> persist results -> cleanup -> schedule expiry
```

Queue messages contain `job_id`, `chunk_id`, `plan_version`, `attempt`, and a
small configuration reference. A consumer claims a pending chunk in a D1
transaction, invokes the provider, writes the normalized segment result, and
marks the chunk complete. Duplicate delivery sees the completed attempt and
returns without a second inference. Retryable failures increment an attempt
record and requeue only that chunk; permanent failures fail the job with an
actionable status. Completion events/counters are durable, so Workflow waits
do not depend on an HTTP request staying open.

Workflow state and large artifacts stay within their documented limits by
storing references and result objects in R2. Queue backpressure and model
rate limits are explicit configuration, not an unbounded fan-out.

## State contracts

Job states:

```text
CREATED -> UPLOADING -> UPLOADED -> ANALYZING -> PREPROCESSING
         -> SPLITTING -> TRANSCRIBING -> MERGING -> POST_PROCESSING
         -> COMPLETED
```

From any active state, authorized cancellation may lead to `CANCELLED`; a
terminal processing error leads to `FAILED`; retention expiry leads to
`EXPIRED`; explicit deletion leads to `DELETED` after content cleanup.
Repositories reject skipped transitions, stale versions, and writes by a
different owner.

Chunk states are `PENDING`, `QUEUED`, `TRANSCRIBING`, `COMPLETED`,
`FAILED_RETRYABLE`, and `FAILED_PERMANENT`. Each transition records an
attempt/version and timestamp. Progress is derived from durable counters and
cannot exceed total chunks or bytes.

## D1 logical model

| Entity | Durable fields (minimum) | Key constraints |
| --- | --- | --- |
| `users` | owner ID, session hash, created/last-seen | session hash unique; no raw session |
| `jobs` | owner, state/version, mode, idempotency key, progress, retention/deleted timestamps | owner+idempotency unique; legal state checks |
| `audio_files` | job, object key, size, MIME, checksum, ffprobe metadata, upload status | one completed original per job |
| `chunks` | job, plan version, interval, object key, state, attempt count | job+plan+index unique; bounded interval |
| `chunk_attempts` | chunk, attempt, status, error class, timestamps | chunk+attempt unique; redact errors |
| `speakers` | job, canonical ID, display name, confidence | job+canonical ID unique; no external identity assumption |
| `transcript_segments` | job/chunk, global start/end, text, speaker, sequence, source version | ordered/indexed; deletion cascades content |

Foreign keys and deletion behavior are explicit in migrations. Large result
text is served from canonical R2 JSON/exports; D1 segment text is retained
only as needed for the transcript API and is deleted with the job.

## Merge and exports

Provider-local timestamps are offset by each chunk start and clamped to source
duration. Segments are sorted by global start/sequence. In overlap windows,
the merge compares normalized Unicode text/tokens, temporal proximity,
ordering, and confidence/speaker hints; deterministic tie-breaking keeps one
canonical segment and records a dedup counter. Identical input and plan
versions produce byte-equivalent canonical JSON. Speaker reconciliation runs
before export, but a rename changes only the canonical speaker display name,
never transcription data.

TXT, Markdown, JSON, SRT, and VTT are generated from that one canonical model.
Timestamp rounding, escaping, line wrapping, UTF-8 encoding, and speaker
labels are versioned and covered by golden tests. Result download authorization
is short-lived and owner-scoped; the bucket is never public.

## Privacy and lifecycle

The success sequence is transactional at the metadata level:

1. Merge and write canonical result objects.
2. Verify result checksums and mark `POST_PROCESSING` complete.
3. Delete the original source and successful temp objects, retrying idempotently.
4. Mark cleanup complete and schedule result expiry.

Failed sources are retained only for the retry window (target 24 hours).
Results are retained at most 7 days. R2 lifecycle rules are defense in depth;
application deletion is required because lifecycle deletion is eventual. A
delete-all request enumerates and removes every `original/`, `temp/`, and
`results/` object for the job and cascades content-bearing D1 rows. Any
remaining tombstone contains only a non-content identifier, reason, and audit
timestamps.

## Security and observability

- Private R2, narrow temporary credentials, strict CORS, and short download
  TTLs protect data at rest and in transit.
- Validate request schemas, file size, extension/MIME, ffprobe output, and
  model limits. Reject path traversal, duplicate keys, unsafe headers, and
  oversized JSON.
- Apply per-session/IP rate limits, abuse quotas, cost guards, and a Turnstile
  hook where risk warrants. CSRF tokens protect cookie-authenticated writes.
- Correlation IDs link Worker, Workflow, Queue, Container, and AI events.
  Logs contain IDs, state, durations, byte counts, and error classes only;
  never transcript text, raw audio, secrets, or complete signed URLs.
- Metrics cover upload throughput/retries, chunk latency/failures, queue
  backlog, model throttles/cost, cleanup lag, and authorization failures.

## Environment and deployment boundaries

Local uses deterministic fakes and Wrangler emulators; test recordings are
synthetic or ignored. DEV resources use a unique prefix and separate account
bindings/secrets. Production names and credentials are never present in local
or DEV files. A production deployment is allowed only after the DEV evidence,
privacy proof, long-audio tests, clean-room build, and release PR gate in
`PLAN.md`.

## Limits and capability evidence

The following official pages were checked on 2026-09-02. Dates below are the
page dates observed during the check; revalidate at implementation time.

| Capability | Observed fact | Design consequence |
| --- | --- | --- |
| R2 multipart | Up to 5 TiB/object, 10,000 parts, 5 MiB-5 GiB parts; resumable | Compute part size; retry parts; abort incomplete uploads |
| R2 presigned URLs | Single GET/PUT/HEAD/DELETE, 1 second-7 day expiry; browser requires CORS | Use only for single operations or per-part signatures; treat as bearer tokens |
| R2 temporary credentials | Short-lived bucket/path-scoped S3 credentials; parent token bounds scope | Prefer for browser multipart; issue least privilege and short TTL |
| Workers AI Whisper Turbo | Batch audio model with structured transcription info/segments/VTT | Normalize response; verify byte limits/timestamp details in capability spike |
| Workers AI limits | Account/task rate limits apply, including local Wrangler inference | Backoff, cap concurrency, and preflight cost/limits |
| Containers | Bounded instance vCPU, memory, and disk; instances are on demand | Stream media and size instance after measurement; never rely on disk persistence |
| Queues | 128 KB messages, up to 100 retries, 15-minute consumer wall time | Metadata-only messages and chunk-sized work |
| Workflows | Durable steps with bounded payload/state/steps/subrequests | Persist large artifacts in R2 and wait durably |
| R2 lifecycle | Deletion is typically within 24 hours; incomplete multipart default rule is 7 days | Keep application cleanup primary and configure a defense rule |
| Static Assets | Worker and assets deploy together; SPA fallback supported | Keep API selective and serve client from the same Worker |

Sources:

- <https://developers.cloudflare.com/r2/objects/upload-objects/index.md>
- <https://developers.cloudflare.com/r2/api/s3/presigned-urls/index.md>
- <https://developers.cloudflare.com/r2/api/s3/temporary-credentials/index.md>
- <https://developers.cloudflare.com/workers-ai/models/whisper-large-v3-turbo/index.md>
- <https://developers.cloudflare.com/workers-ai/platform/limits/index.md>
- <https://developers.cloudflare.com/containers/platform/limits/index.md>
- <https://developers.cloudflare.com/queues/platform/limits/index.md>
- <https://developers.cloudflare.com/workflows/reference/limits/index.md>
- <https://developers.cloudflare.com/r2/buckets/object-lifecycles/index.md>
- <https://developers.cloudflare.com/workers/static-assets/index.md>

Open gates are tracked in `PLAN.md`: exact browser temporary-credential
semantics, model input/timestamp limits, Container networking/resource
behavior, and Cloudflare-only diarization support for Chinese/mixed language.
