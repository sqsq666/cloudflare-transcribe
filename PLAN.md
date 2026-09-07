# Delivery Plan

This plan is authoritative for autonomous development. Status markers:

- `[x]` complete and verified
- `[>]` next task / in progress
- `[ ]` pending
- `[!]` gated on an explicit external capability or decision

The plan is deliberately local-first. No production resource is created by
the development loop, and a DEV resource is not created until Phase 15.

## Architecture baseline

The product is a Cloudflare Worker application with a static browser client.
The Worker is a control plane: it authenticates a browser session, creates
jobs, issues narrowly scoped upload/download authorization, records metadata,
and exposes status/transcript APIs. It never proxies a large audio request
body.

The browser uploads directly to a private R2 bucket. Small objects may use a
single signed PUT; objects above a configurable threshold use S3-compatible
multipart upload with short-lived, path-scoped R2 temporary credentials (or
equivalent per-part signed operations if a capability review proves that is
the better current option). The client persists upload ID and part ETags for
resume. Completion is a metadata-only Worker call; the Worker verifies the
completed object with R2 before starting processing.

D1 stores structured metadata only: users/session owners, jobs, audio files,
chunks, attempts, speakers, and transcript segments. All mutations go through
repositories that enforce ownership, legal state transitions, constraints,
indexes, and idempotency keys.

One recording maps to one durable Workflow instance. Workflow steps analyze
and plan the recording, invoke an ephemeral Container for ffprobe/FFmpeg, and
coordinate chunk work. Queues carry chunk IDs and bounded metadata to workers
that fetch a temporary R2 chunk reference and call the Workers AI adapter.
Chunk completion is persisted transactionally; retries are independent and
safe to repeat. Workflow waits use durable events or bounded sleeps, never a
long-lived request connection.

The standard provider is fixed at
`@cf/openai/whisper-large-v3-turbo`. The adapter normalizes the model's
structured segment response into the canonical transcript contract. A
model-limit-aware planner chooses encoded chunk size/duration and 5-15 second
overlap based on ffprobe metadata and the capability configuration. Container
disk is ephemeral `/tmp` scratch; source and intermediate objects are streamed
and removed promptly.

The merge service offsets chunk-local timestamps, removes overlap duplicates
using time windows plus normalized token similarity and ordering, and runs a
speaker reconciliation layer. Canonical segments drive D1, R2 result files
(`transcript.txt`, `.md`, `.json`, `.srt`, `.vtt`), and the transcript UI.
Speaker diarization is behind a provider interface and is enabled in V1 only
after current Cloudflare-hosted support for the required languages is proved.

Successful jobs persist results, delete source and successful temp objects,
and expose immediate deletion. Failed source retention targets 24 hours and
results target 7 days; application cleanup and R2 lifecycle rules are both
required. No content-bearing tombstone may remain after an explicit delete.

## Environment policy

| Environment | Purpose | Durable services | Rules |
| --- | --- | --- | --- |
| Local | Unit/integration/browser tests and FFmpeg fixtures | Wrangler local D1/R2 emulators or deterministic fakes | No external credentials; no user recordings committed |
| DEV | Isolated end-to-end validation | Clearly named DEV Worker, R2, D1, Queue, Workflow, Container, Workers AI | Requires valid DEV credentials; never points at production; disposable test data |
| Production | Release deployment only | Cloudflare-only resources with separate names and secrets | No VPS or workstation dependency; deploy only after release gate and human merge |

Every binding, bucket, database, queue, workflow, and container image must
carry an environment-specific name. Secrets are supplied by local secret
stores or Wrangler, never checked in.

## Ordered phases and tasks

### Phase 1 - architecture and contracts (planning checkpoint)

Dependencies: `PROJECT_SPEC.md` and current official Cloudflare docs.

- `[x] 1.1` Inspect the specification, repository, branch, worktree, and
  recent history; confirm the `dev` branch and zero-VPS constraint.
- `[x] 1.2` Record the component/data-flow baseline, environment boundaries,
  privacy invariants, and unresolved Cloudflare capability gates in
  `docs/architecture.md`.
- `[x] 1.3` Add ADRs for Cloudflare-only storage/orchestration, direct scoped
  R2 upload authorization, and the diarization boundary; add `AGENTS.md`,
  this plan, and `STATE.md`.

Acceptance: a fresh agent can identify the next task without conversation
history; architecture prohibits Worker-proxied large uploads, persistent
container data, non-Cloudflare production dependencies, and secrets in Git;
all external assumptions have an official source or an explicit gate.

### Phase 2 - project skeleton and local tooling

Dependencies: Phase 1. No Cloudflare account is required.

- `[ ] 2.1` Create a TypeScript Worker plus Workers Static Assets/Vite client
  skeleton, package manifest, lockfile, strict compiler settings, and scripts
  for build, typecheck, lint, format, and test.
- `[ ] 2.2` Add Wrangler local configuration with mock/fake bindings and
  environment overlays. Keep DEV and production configuration separate and
  ignored values out of the repository.
- `[ ] 2.3` Add a health route, static SPA fallback, structured error format,
  request correlation ID, and a minimal test harness.
- `[ ] 2.4` Add CI configuration that runs secret scanning, formatting check,
  lint, typecheck, unit tests, and a production-build check without secrets.

Acceptance: `npm` (or the selected package manager) install, typecheck, lint,
test, and build all pass on a clean machine; `wrangler dev` serves a health
route and the static shell locally; no external service is contacted by unit
tests.

### Phase 3 - domain contracts, D1 schema, and state machines

Dependencies: Phase 2.

- `[ ] 3.1` Define versioned TypeScript contracts for jobs, audio files,
  chunks, attempts, speakers, segments, progress events, API errors, and
  idempotency keys.
- `[ ] 3.2` Write ordered D1 migrations for `users`, `jobs`, `audio_files`,
  `chunks`, `chunk_attempts`, `speakers`, and `transcript_segments`, including
  foreign keys, check constraints, uniqueness, timestamps, ownership indexes,
  progress indexes, and retention fields.
- `[ ] 3.3` Implement repository interfaces and a local D1 adapter. Keep large
  text/audio artifacts in R2 and store only bounded metadata or canonical
  segment text according to the documented retention policy.
- `[ ] 3.4` Implement strict job and chunk transition functions. Reject
  illegal transitions, stale versions, duplicate completions, and ownership
  mismatches transactionally.
- `[ ] 3.5` Add deterministic tests for all transitions, concurrent update
  races, retry counters, idempotent replay, and migration integrity.

Acceptance: migrations apply/rollback in local test databases; invalid state
changes fail without partial writes; replaying an operation has one effect;
queries are indexed by owner/job/chunk; no schema column stores raw audio.

### Phase 4 - identity and Worker control API

Dependencies: Phase 3.

- `[ ] 4.1` Implement an injectable identity/session adapter. The local/V1
  baseline uses an opaque, cryptographically random HttpOnly session cookie
  mapped to a `users` row; account federation remains outside V1 unless an
  ADR selects a Cloudflare-native provider.
- `[ ] 4.2` Implement authenticated job creation with format, size, mode, and
  idempotency validation; never trust client owner IDs.
- `[ ] 4.3` Implement upload-session create/resume/abort/complete control
  endpoints, status/progress endpoints, transcript/result authorization,
  speaker rename, and delete-all-data-now endpoints.
- `[ ] 4.4` Define bounded JSON schemas, consistent HTTP errors, rate-limit
  hooks, CORS policy, and request correlation/logging behavior.

Acceptance: every endpoint has ownership tests and bounded request parsing;
cross-session access is denied; complete cannot start processing until R2
metadata verifies the expected object; delete is authenticated and idempotent.

### Phase 5 - responsive client and upload abstraction

Dependencies: Phase 4 for contracts; can use fake storage.

- `[ ] 5.1` Build the responsive upload page with all seven required format
  labels, file name/size, mode control, progress percentage/speed, task state,
  chunk progress, retry/error states, and completion notification.
- `[ ] 5.2` Implement a storage-provider interface and deterministic local
  fake that models part retries, resume, abort, checksums/ETags, and network
  interruption without storing private fixtures in Git.
- `[ ] 5.3` Implement browser upload persistence (IndexedDB or equivalent)
  for upload ID and completed parts, bounded parallelism, cancellation, and
  restart recovery. Keep progress based on bytes acknowledged by R2.
- `[ ] 5.4` Add transcript view shell with timestamps, search, copy, speaker
  filter/rename, exports, deletion confirmation, and accessible mobile/desktop
  layout.

Acceptance: Playwright or component tests cover keyboard/mobile/desktop
states, all upload error paths, resume after reload, and no UI state implies
completion before server confirmation; controls do not send audio to the
Worker API.

### Phase 6 - direct R2 multipart protocol

Dependencies: Phase 5 and the R2 capability gate in Phase 15; local fake may
land first.

- `[ ] 6.1` Implement a server-side authorization adapter that mints short-TTL
  temporary R2 credentials or per-operation signatures scoped to one bucket
  and `original/{job_id}/...`; never return a parent token.
- `[ ] 6.2` Implement browser S3 multipart create/upload-part/list/complete and
  abort flows with CORS, part ETag recording, retries, and resume. Choose a
  configurable threshold and compute part size so the 10,000-part limit is
  respected; enforce the 5 MiB minimum for non-final parts.
- `[ ] 6.3` Verify one completed object at
  `original/{job_id}/source.<validated-extension>`, expected byte size,
  content type, and owner before transitioning `UPLOADING -> UPLOADED`.
- `[ ] 6.4` Add incomplete-upload reconciliation and a lifecycle defense rule;
  do not treat an abandoned multipart upload as a completed object.

Acceptance: integration tests prove large bytes bypass Worker request bodies,
failed parts alone retry, reload resumes, malformed ETags/part numbers are
rejected, unauthorized prefixes fail, and exactly one completed original
object exists after completion. R2 CORS is restricted to the application
origin.

### Phase 7 - local media analysis and streaming processor

Dependencies: Phase 3, 6. Local FFmpeg is allowed on the development host;
production execution remains a Container adapter.

- `[ ] 7.1` Add a pinned FFmpeg/ffprobe Container image and a local command
  adapter. Document licensing, version, checksums, and update procedure.
- `[ ] 7.2` Validate extension/MIME and run ffprobe safely to obtain duration,
  codec, sample rate, channels, bitrate, container format, and byte size.
- `[ ] 7.3` Implement streaming source-to-output processing using bounded
  `/tmp` files or pipes, avoiding whole-recording PCM WAV intermediates. Keep
  source/chunk/result paths out of persistent volumes.
- `[ ] 7.4` Emit manageable encoded chunk objects and clean them in `finally`
  blocks even after cancellation or failure.
- `[ ] 7.5` Add fixtures for M4A (including Apple Voice Memo), MP3, WAV, FLAC,
  OGG, WebM, and AAC plus malformed/hostile metadata cases.

Acceptance: metadata is complete and validated; supported fixtures process;
resource tests show bounded scratch use; a simulated restart leaves durable
state recoverable and no required artifact only on local disk.

### Phase 8 - adaptive chunk planner

Dependencies: Phase 7 and current Workers AI capability data.

- `[ ] 8.1` Define a versioned model-capability configuration containing input
  encoding, byte/request limits, maximum safe duration, concurrency, and
  timeout assumptions. Fail closed when required limits are unknown.
- `[ ] 8.2` Plan chunks from duration, codec, sample rate, estimated encoded
  bytes, speech density, and model limits; target 5-15 minutes only as a
  starting point, not a fixed duration.
- `[ ] 8.3` Add 5-15 second overlap with explicit first/last-boundary rules,
  monotonically increasing global offsets, and a maximum chunk count guard.
- `[ ] 8.4` Add property/golden tests for short files, silent files, highly
  compressed/uncompressed media, 2-hour/4-hour recordings, and 1GB+ inputs.

Acceptance: every planned chunk fits the active capability envelope (or is
rejected with a useful error), covers the source exactly once apart from
documented overlap, has deterministic IDs, and can be replanned idempotently.

### Phase 9 - AI provider contract and capability verification

Dependencies: Phase 8; local mock first.

- `[ ] 9.1` Define an AI provider interface accepting a chunk reference and
  returning normalized segments, language metadata, confidence where
  available, and provider/model/version diagnostics.
- `[ ] 9.2` Implement a deterministic fake provider for tests and a Workers AI
  adapter fixed to `@cf/openai/whisper-large-v3-turbo`. Keep audio payloads
  bounded and fetched from R2 only inside the processing path.
- `[ ] 9.3` Run a documented capability spike against current official docs and
  DEV only when authorized: model input shape/size, timestamp granularity,
  Chinese, English, and mixed-language behavior, rate limits, and error
  classes. Update the planner config from verified facts.
- `[!] 9.4` Verify a Cloudflare-hosted diarization provider for Chinese and
  mixed-language audio. Until verified, retain speaker data structures and
  UI architecture but disable/clearly label diarization; never add a
  non-Cloudflare production provider silently.

Acceptance: mock tests are deterministic; the adapter maps model responses
without dropping timestamps; unsupported/oversized input fails before billable
inference; evidence for diarization is recorded as enabled or explicitly
unsupported.

### Phase 10 - durable Workflow and Queue orchestration

Dependencies: Phases 3, 6-9.

- `[ ] 10.1` Define the job Workflow and idempotent step keys:
  `validate-upload`, `analyze`, `plan`, `enqueue`, `await-chunks`, `merge`,
  `export`, `cleanup`, and `expire`.
- `[ ] 10.2` Define a Queue message schema containing only job/chunk IDs,
  attempt/version, and bounded configuration references. Enforce backpressure,
  retry limits, dead-letter handling, and no audio payloads.
- `[ ] 10.3` Implement chunk consumers that claim work transactionally,
  invoke the provider, persist success/failure, and release/cleanup temp
  objects. A replay of the same message must not duplicate segments or spend
  an unnecessary inference.
- `[ ] 10.4` Implement durable progress counters and completion signaling via
  events or bounded sleeps. Recover safely after Worker, Queue, Workflow, or
  Container restarts.
- `[ ] 10.5` Add local orchestration fakes and failure injection for timeout,
  retryable/permanent errors, duplicate delivery, out-of-order completion,
  and partial job recovery.

Acceptance: a multi-chunk job can lose and retry one chunk without restarting
others; progress is monotonic and truthful; all Workflow/Queue payloads stay
within documented limits; no request is held open for the job lifetime.

### Phase 11 - merge, speaker reconciliation, and canonical transcript

Dependencies: Phase 9 and planner output.

- `[ ] 11.1` Normalize provider segments and convert local timestamps to global
  timestamps with millisecond precision and source-duration clamping.
- `[ ] 11.2` Deduplicate overlap using temporal windows, normalized Unicode
  text/token similarity, segment ordering, and confidence/speaker hints. Keep
  a deterministic tie-break and audit counters.
- `[ ] 11.3` Reconcile chunk-local speaker labels into canonical speaker IDs
  using overlap evidence and documented confidence; preserve uncertainty
  instead of inventing identity.
- `[ ] 11.4` Persist canonical segments transactionally and make merge replay
  idempotent. Reject missing/duplicate chunk versions.
- `[ ] 11.5` Add fixtures for Chinese, English, mixed-language, punctuation,
  silence, repeated phrases, boundary speech, and conflicting speakers.

Acceptance: merged output has no overlap duplicates in golden fixtures, global
timestamps are ordered and bounded, speaker renames affect display data only,
and rerunning merge yields byte-equivalent canonical JSON.

### Phase 12 - exports and transcript experience

Dependencies: Phase 11 and Phase 5 UI shell.

- `[ ] 12.1` Generate TXT, Markdown, JSON, SRT, and VTT from one canonical
  transcript model. Define escaping, encoding, timestamp rounding, and
  speaker-label conventions.
- `[ ] 12.2` Store result objects under `results/{job_id}/` with content type,
  checksum, schema/version metadata, and expiry timestamp.
- `[ ] 12.3` Authorize downloads through short-lived access or Worker streams
  of result-sized artifacts; never expose a bucket publicly.
- `[ ] 12.4` Wire search, copy, speaker filtering/rename, export selection,
  progress refresh, and completion notification to the API contract.

Acceptance: exporter golden tests agree across all formats; malformed data
cannot produce unsafe headers; downloads are owner-scoped and expire; the UI
never displays stale speaker names after a successful rename.

### Phase 13 - privacy cleanup, expiry, and deletion

Dependencies: Phases 6, 10-12.

- `[ ] 13.1` Implement success cleanup: persist results first, then delete the
  original and successful temp objects, and record cleanup completion.
- `[ ] 13.2` Implement retry-window cleanup for failed source objects (target
  24 hours), result expiry (target 7 days), and idempotent scheduled scans.
- `[ ] 13.3` Configure R2 lifecycle rules for result prefixes and incomplete
  multipart uploads as defense in depth; document the expected eventual
  deletion window.
- `[ ] 13.4` Implement and test authenticated delete-all-data-now across R2
  original/temp/results and D1 chunks, attempts, segments, speakers, and
  content-bearing job records. Allow only a content-free tombstone when
  necessary for abuse/audit controls.

Acceptance: deletion is complete after retries, source/temp objects are absent
after successful completion, failed-source and result retention bounds are
enforced by both app logic and lifecycle configuration, and no transcript text
or audio survives an explicit delete.

### Phase 14 - security, abuse controls, and observability

Dependencies: Phases 4-13.

- `[ ] 14.1` Enforce private R2, unpredictable IDs, ownership on every route,
  signed/temporary access, upload byte/count limits, extension/MIME/ffprobe
  validation, and safe content-disposition headers.
- `[ ] 14.2` Add rate limiting, abuse quotas/cost guards, optional Turnstile
  hook, request size limits, CSRF protections for cookie-authenticated writes,
  and denial telemetry without logging content.
- `[ ] 14.3` Add structured logs, metrics/traces, correlation IDs, state-change
  audit events, Queue/Workflow diagnostics, and alerts for cleanup failures or
  backlog growth. Redact credentials, URLs with signatures, and transcript
  text.
- `[ ] 14.4` Run dependency/license review, static analysis, secret scanning,
  and an authorization/privacy test matrix.

Acceptance: threat-model findings have mitigations or explicit residual-risk
entries; cross-user, replay, expiry, abuse, and malformed-media tests pass;
observability can diagnose a job by ID without revealing user content.

### Phase 15 - isolated Cloudflare DEV integration gate

Dependencies: Phases 1-14 local acceptance; valid DEV credentials; no
production changes.

Before provisioning, verify current official documentation and account limits
for each capability. Record resource names, dates, and rollback/deletion
commands in `docs/dev-runbook.md` without recording secrets.

- `[ ] 15.1` Confirm DEV account/permissions, billing limits, region/model
  availability, and a secret-injection method. If credentials are absent,
  leave this task pending and continue local tasks; do not create `BLOCKED`.
- `[ ] 15.2` Provision clearly prefixed DEV D1 and private R2 with CORS and
  lifecycle rules; apply migrations and verify ownership isolation.
- `[ ] 15.3` Provision DEV Worker/Static Assets and Workers AI binding; run a
  minimal model capability check with synthetic/non-sensitive audio.
- `[ ] 15.4` Provision DEV Container image and verify ffprobe/FFmpeg resource
  behavior, `/tmp` cleanup, network access, and restart recovery.
- `[ ] 15.5` Provision DEV Queue and Workflow bindings, then run a one-chunk
  smoke job before enabling multi-hour tests.
- `[ ] 15.6` Enable real browser multipart upload using only short-lived scoped
  R2 credentials. Verify no permanent token reaches browser devtools/logs.

DEV gate acceptance: all resources are environment-separated; smoke upload,
analysis, inference, merge, export, cleanup, and deletion pass; resource
limits and costs are recorded; production IDs/secrets are never used.

### Phase 16 - DEV end-to-end and resilience verification

Dependencies: Phase 15.

- `[ ] 16.1` Run Playwright desktop/mobile flows for every supported format,
  progress, reload/resume, transcript search/copy/filter/rename, exports, and
  delete-all.
- `[ ] 16.2` Run representative 2-hour, 4-hour, and 1GB+ recordings (or
  deterministic size/duration equivalents where account limits prohibit full
  fixtures). Measure chunk count, memory, disk, latency, retries, and cost.
- `[ ] 16.3` Inject part loss, Queue redelivery, provider throttling, Container
  restart, Workflow replay, and browser disconnect; verify independent
  recovery and truthful progress.
- `[ ] 16.4` Verify privacy windows, lifecycle cleanup, signed download expiry,
  no residual source/temp data, and observability redaction in DEV logs.

Acceptance: all V1 scenarios pass in isolated DEV with evidence retained in
non-sensitive test reports; failures are fixed or documented as a real
external blocker with `.codex-agent/BLOCKED` only when human credentials or a
decision is unavoidable.

### Phase 17 - production readiness and release handoff

Dependencies: every prior acceptance criterion, including DEV evidence.

- `[ ] 17.1` Freeze and review architecture, limits snapshot, threat model,
  privacy/deletion proof, cost controls, runbooks, and rollback procedure.
- `[ ] 17.2` Run clean-room install, lint, typecheck, build, unit/integration
  tests, secret scan, and final dependency audit.
- `[ ] 17.3` Confirm production configuration is separate, least-privilege,
  and has no VPS/container-host dependency; do not deploy from this loop.
- `[ ] 17.4` Create or update a release-readiness pull request from `dev` to
  `main`, link verification evidence, and document that merge/deployment
  requires the appropriate human release action.

Production gate acceptance: V1 criteria are demonstrably complete, long-audio
and privacy behavior are verified, docs are current, `dev` is pushed, and the
release PR exists. Do not create `.codex-agent/DONE` before this gate.

## Dependency map

```text
1 architecture
  -> 2 skeleton -> 3 schema/state -> 4 API/identity
  -> 5 client/upload fake -> 6 direct R2
  -> 7 media -> 8 planner -> 9 AI contract
  -> 10 Workflow/Queue -> 11 merge -> 12 exports/UI
  -> 13 cleanup -> 14 security/observability
  -> 15 DEV resources/integration -> 16 DEV E2E
  -> 17 production readiness/PR
```

Phases 7-9 may proceed in parallel after their input contracts exist, but
orchestration cannot be accepted until all three are complete. Security and
privacy tests may begin earlier and are required before DEV.

## Cross-cutting acceptance criteria

V1 is accepted only when all of the following are true:

1. Browser uploads all required formats directly to a private R2 bucket,
   including resumable multipart uploads for 1GB+ objects, with no large
   Worker request body and exactly one completed original object.
2. Ownership, authorization, validation, rate/abuse controls, and safe signed
   access are enforced server-side; no secret is exposed or committed.
3. ffprobe metadata is complete; FFmpeg processing is streaming-oriented and
   uses bounded ephemeral scratch space.
4. Chunk planning is adaptive to verified model/encoding limits, uses overlap,
   and is restart/idempotency safe for multi-hour recordings.
5. Standard mode uses the fixed Whisper Turbo model with verified multilingual
   segment/timestamp behavior. Speaker data/reconciliation exists, and
   diarization is enabled only with a verified Cloudflare-only capability.
6. Queue/Workflow retries are independent at chunk level, progress is
   durable/truthful, and replay does not duplicate work or text.
7. Canonical merged output has global ordered timestamps and no overlap
   duplicates; all five exports and the transcript UI derive from it.
8. Successful processing deletes source and successful temporary audio; failed
   source retention is at most the documented window, results at most seven
   days, and immediate delete removes all content-bearing data.
9. Typecheck, lint, build, unit/integration/security/privacy tests, and DEV
   E2E/long-audio/recovery tests pass with evidence.
10. Architecture and operations remain Cloudflare-only in production and the
    release-readiness PR is open from `dev` to `main`.

## Testing expectations by risk

| Risk | Required evidence |
| --- | --- |
| State corruption/races | Transaction tests, transition table coverage, idempotent replay and stale-version tests |
| Large uploads | Browser integration with interrupted parts, reload/resume, 1GB+ synthetic object, no Worker-body assertion |
| Media diversity | ffprobe/FFmpeg fixtures for all formats, malformed input rejection, scratch-space measurement |
| Model limits/cost | Capability snapshot, preflight rejection, bounded chunk property tests, throttling/backoff tests |
| Retry/restart | Queue duplicate delivery, Workflow replay, Container loss, partial chunk completion and recovery |
| Merge quality | Timestamp/overlap/multilingual/speaker golden fixtures and deterministic output hashes |
| Privacy/security | Cross-owner matrix, signed URL expiry, deletion/list verification, retention/lifecycle checks, redacted logs |
| UX | Playwright desktop/mobile keyboard and network-failure flows |

## DEV integration gates and blockers

Cloudflare DEV work starts only after local phases are green and a current
documentation snapshot is recorded. Missing DEV credentials are an expected
pending gate, not a code blocker: keep implementing local adapters/tests.
Create `.codex-agent/BLOCKED` only if a required external credential or
unavoidable account decision is reached and cannot be safely worked around.
Transient API, model, registry, or GitHub failures remain retryable.

## Production gate

Production deployment is outside the autonomous loop. The agent may prepare
configuration, runbooks, evidence, and a `dev` -> `main` pull request after
Phase 17, but must not merge the PR, deploy production, or modify production
resources without a later explicit release action.

## Official documentation snapshot used for planning

Checked 2026-09-02 (URLs and dates are revalidated at implementation gates):

- R2 upload objects: <https://developers.cloudflare.com/r2/objects/upload-objects/index.md>
  (updated 2026-07-29): multipart max object 5 TiB, up to 10,000 parts, part
  size 5 MiB-5 GiB, resumable; incomplete uploads should be aborted.
- R2 presigned URLs: <https://developers.cloudflare.com/r2/api/s3/presigned-urls/index.md>
  (updated 2026-08-22): single-operation GET/PUT/HEAD/DELETE, expiry 1 second
  to 7 days, browser use requires CORS, and URLs are bearer tokens.
- R2 temporary credentials:
  <https://developers.cloudflare.com/r2/api/s3/temporary-credentials/index.md>
  (updated 2026-04-24): short-lived bucket/path-scoped S3 credentials;
  explicit action scoping is currently local-signing only; never expose the
  parent token.
- Workers AI model:
  <https://developers.cloudflare.com/workers-ai/models/whisper-large-v3-turbo/index.md>
  (model page checked 2026-09-02): fixed model ID, batch capability, audio
  input, structured transcription info/segments/VTT; input limits and actual
  timestamp behavior still require a capability spike.
- Workers AI limits: <https://developers.cloudflare.com/workers-ai/platform/limits/index.md>
  (updated 2026-08-07): account/task rate limits apply, including local
  Wrangler inference; planner and backoff must use current values.
- Containers limits:
  <https://developers.cloudflare.com/containers/platform/limits/index.md>
  (updated 2026-08-28): predefined instances provide bounded vCPU, memory,
  and ephemeral disk; select a type only after measuring FFmpeg workload.
- Queues limits: <https://developers.cloudflare.com/queues/platform/limits/index.md>
  (updated 2026-04-21): 128 KB messages, 100 retries, 15-minute consumer wall
  time, and bounded throughput/retention; messages must remain metadata-only.
- Workflows limits: <https://developers.cloudflare.com/workflows/reference/limits/index.md>
  (updated 2026-06-15): durable per-step execution, bounded payload/state,
  step/concurrency/subrequest limits; large artifacts belong in R2.
- R2 lifecycle: <https://developers.cloudflare.com/r2/buckets/object-lifecycles/index.md>
  (updated 2026-04-21): deletion is typically within 24 hours and incomplete
  multipart uploads have a seven-day default rule; application cleanup remains
  primary for the privacy promise.
- Workers Static Assets:
  <https://developers.cloudflare.com/workers/static-assets/index.md>
  (updated 2026-07-03): Worker and assets deploy as one unit; SPA fallback and
  selective Worker-first routing are supported.

These facts are planning inputs, not permanent constants. Recheck before any
implementation that depends on a limit, API shape, model behavior, or pricing.

