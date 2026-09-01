# Cloudflare-only Audio Transcription Platform

## Absolute architecture rule

The final production platform must be Cloudflare-only.

ZERO VPS.

The development host may be used for development, testing, Codex, Git,
Docker and local FFmpeg testing, but the deployed production application
must never depend on this host or any traditional server.

Production must not depend on:

- VPS
- persistent Linux server
- Nginx
- Docker VPS
- Docker Compose server
- MySQL server
- PostgreSQL server
- Redis server
- PM2
- Supervisor
- Certbot
- persistent local Container storage

Cloudflare Container filesystem is ephemeral scratch space only.

Persistent application state must live in Cloudflare services such as R2
and D1.

---

## Product

Users upload audio recordings from a browser and receive transcripts.

Supported formats must include:

- M4A
- MP3
- WAV
- FLAC
- OGG
- WebM
- AAC

Apple Voice Memo M4A is an important target.

The system must support:

- short recordings
- tens of minutes
- 2 hour recordings
- 4 hour recordings
- longer recordings
- hundreds of MB
- 1GB+ files

Users must not manually convert, compress or split files.

---

## Cloudflare architecture

Use Cloudflare-first capabilities:

- Workers
- Workers Static Assets
- R2
- D1
- Workflows
- Queues
- Containers
- Workers AI
- Turnstile where useful
- Cloudflare rate limiting/security
- Cloudflare Observability

Before implementing capabilities that depend on limits or model behavior,
check the CURRENT official Cloudflare documentation.

Do not rely on old tutorials.

---

## Frontend

V1 frontend should be clean and responsive on desktop and mobile.

Core upload page:

Audio Transcription

Drop your audio recording here

M4A · MP3 · WAV · FLAC · OGG · WebM · AAC

[ Select Audio ]

Transcription mode:

Standard
Speaker Detection

Display:

- file name
- file size
- upload percentage
- upload speed
- task status
- transcription progress
- completed chunks / total chunks
- completion notification

---

## Upload architecture

Large files MUST NOT pass through a Worker request body.

Required architecture:

Browser
→ Worker creates job / grants upload authorization
→ Browser uploads directly to private R2
→ Multipart Upload for large files
→ upload completion confirmation

Upload chunks and AI audio chunks are different concepts.

R2 must contain one completed original audio object after Multipart
completion.

Example:

original/{job_id}/source.m4a

Support:

- Multipart Upload
- upload progress
- part retries
- resumability where practical
- incomplete upload cleanup
- private R2 bucket

Prefer current Cloudflare-supported secure temporary/scoped credentials or
equivalent current official mechanism.

Never expose permanent high-privilege R2 credentials to the browser.

---

## Audio analysis

After upload, start durable processing.

Use Cloudflare Containers for FFmpeg and ffprobe.

ffprobe must extract useful metadata including:

- duration
- codec
- sample rate
- channels
- bitrate
- file size
- container format

The system decides automatically whether the file can be sent directly or
must be split.

---

## Container rules

Container disk is ephemeral only.

Use paths such as:

/tmp/input
/tmp/chunk-0001

Never persist:

- source audio
- transcript
- database
- user content
- configuration

on Container local disk.

Container restart/destruction must not cause loss of durable job state.

---

## Audio preprocessing

Avoid huge intermediate PCM WAV files.

Prefer streaming-oriented processing:

R2 input
→ FFmpeg
→ generate a manageable AI chunk
→ upload/send chunk
→ remove local temporary data
→ continue

Use a compatible efficient intermediate audio format such as FLAC where
appropriate.

Do not unnecessarily transcode compatible input.

---

## Dynamic audio chunking

Do not hard-code one fixed chunk duration.

Chunk planning must consider current model limits, duration, encoded size,
codec and other real constraints.

Initial target may be approximately 5–15 minutes, but the planner must be
able to adjust.

Chunks must overlap enough to protect sentence boundaries.

Typical overlap target:

5–15 seconds

Example:

Chunk A 00:00 → 10:10
Chunk B 10:00 → 20:10

---

## Standard transcription model

V1 Standard mode is fixed to:

@cf/openai/whisper-large-v3-turbo

Do not add unnecessary model selection UI for V1.

Target capabilities:

- automatic language handling
- Chinese
- English
- mixed Chinese/English where supported
- timestamps
- segments

---

## Speaker Detection

The product architecture must support diarization.

Do not assume speaker IDs are stable across independently transcribed
chunks.

Implement a speaker reconciliation layer.

Before finalizing the V1 diarization provider, verify CURRENT official
Cloudflare-hosted model support, especially Chinese and mixed-language
speaker diarization.

Do not silently introduce a non-Cloudflare dependency in production.

If a Cloudflare-only requirement cannot currently be satisfied, document
the limitation clearly rather than violating the architecture.

---

## Transcript merging

Chunk-local timestamps must be converted to global timestamps.

Overlap duplicates must be removed using a combination of:

- timestamps
- overlap windows
- normalized text/token similarity
- segment ordering
- speaker information where available

Users must not see duplicated text caused by chunk overlap.

---

## Job lifecycle

Use a durable job state model similar to:

CREATED
UPLOADING
UPLOADED
ANALYZING
PREPROCESSING
SPLITTING
TRANSCRIBING
MERGING
POST_PROCESSING
COMPLETED
FAILED
CANCELLED
EXPIRED
DELETED

Invalid state transitions must be rejected.

---

## Chunk lifecycle

Chunks should have independent states such as:

PENDING
QUEUED
TRANSCRIBING
COMPLETED
FAILED_RETRYABLE
FAILED_PERMANENT

A failed chunk must be retried independently.

Never restart an entire multi-hour recording because one chunk failed.

All processing must be idempotent.

---

## Durable orchestration

One recording corresponds to a durable Workflow job.

Use Workflows for job-level orchestration.

Use Queues for independent chunk-level transcription and retry/concurrency
control where appropriate.

Queue messages must contain metadata/IDs, not audio payloads.

---

## D1

D1 stores structured metadata only.

Expected logical entities include:

- users
- jobs
- audio_files
- chunks
- chunk_attempts
- speakers
- transcript_segments

Do not store large audio objects in D1.

Add proper constraints and indexes.

Use migrations.

---

## R2 structure

Use a private R2 bucket.

Logical prefixes:

original/{job_id}/...
temp/{job_id}/...
results/{job_id}/...

Result files:

transcript.txt
transcript.md
transcript.json
transcript.srt
transcript.vtt

---

## Privacy and deletion

Core promise:

ZERO PERMANENT AUDIO STORAGE.

On successful completion:

1. Persist the completed transcript/results.
2. Delete original source audio promptly.
3. Delete successful temporary chunks immediately.
4. Ensure temp/{job_id} is empty by workflow completion.

Failed jobs may retain source audio only for short retry windows.

Target maximum failed-job source retention:

24 hours unless a documented technical reason requires otherwise.

Transcription results:

maximum 7 days.

Use R2 Lifecycle as cleanup defense-in-depth.

Do not rely exclusively on scheduled application deletion.

---

## Delete all data now

Completed job UI must offer:

Delete all data now

This must actually remove relevant:

- R2 original objects
- R2 temp objects
- R2 result objects
- transcript segments
- speakers
- chunks
- other content-bearing job records

A minimal privacy-safe job tombstone may remain only if deliberately
required.

Do not retain transcript text or audio after deletion.

---

## Transcript page

Provide:

- timestamps
- transcript text
- search
- copy
- speaker filters
- speaker rename

Renaming a speaker must immediately affect display without retranscribing.

---

## Export

Generate:

- TXT
- Markdown
- JSON
- SRT
- VTT

Exports must use the canonical merged transcript.

---

## Security

Implement appropriate:

- private R2
- job ownership checks
- unpredictable IDs
- signed/temporary access
- upload size limits
- MIME validation
- extension validation
- ffprobe validation
- rate limiting
- Turnstile where appropriate
- abuse controls
- safe download authorization

Never commit secrets.

---

## Repository security

This repository is public.

NEVER commit:

- API keys
- GitHub credentials
- Cloudflare tokens
- session secrets
- .env files
- .dev.vars
- credential files
- private user data

Create and maintain a strict .gitignore.

If a secret is accidentally staged or committed, stop and remediate before
pushing.

---

## Development phases

Development must be Local-first.

Do not create production Cloudflare resources during early development.

Expected progression:

1. architecture/contracts
2. project skeleton
3. local D1/schema/repositories
4. job/chunk state machines
5. frontend mock UI
6. upload abstraction
7. local FFmpeg/ffprobe processor
8. dynamic chunk planner
9. AI provider abstraction
10. transcript merge/dedup
11. exporters
12. cleanup engine
13. tests
14. Cloudflare DEV resources
15. real R2 multipart integration
16. real Workers AI integration
17. real Cloudflare Container integration
18. Queues + Workflows integration
19. full DEV E2E
20. large-file/long-recording tests
21. failure-recovery tests
22. privacy/security verification
23. production preparation only after DEV is verified

---

## Cloudflare environments

Development resources must be clearly named/scope-separated from eventual
production resources.

Production must never be modified casually by the autonomous development
agent.

The agent may create DEV resources only when development reaches the
integration phase and valid DEV Cloudflare credentials are available.

If credentials are absent, complete every possible local phase first.

---

## V1

V1 includes:

- browser upload
- large multipart upload
- supported audio formats
- automatic metadata detection
- automatic preprocessing
- automatic dynamic chunking
- Standard Whisper Large V3 Turbo transcription
- independent chunk retries
- merge/dedup
- transcript page
- exports
- progress
- speaker data architecture
- diarization only where Cloudflare-only current capability is verified
- cleanup
- seven-day results expiry
- immediate delete
- security basics
- mobile/desktop responsiveness

---

## V2

Do not allow V2 scope to block V1.

Possible V2 features include:

- AI summaries
- key points
- action items
- assignment requirements extraction
- translation
- transcript editing
- synchronized audio/text playback
- realtime microphone transcription
- notifications
- billing
- advanced speaker identity reconciliation

---

## Development quality rules

- TypeScript where appropriate
- complete runnable code
- no fake implementation pretending to work
- no placeholder TODOs in completed milestones unless explicitly documented
- tests for important state machines and merge logic
- lint/typecheck/test before milestone completion
- idempotency for retryable operations
- useful errors
- observability
- current Cloudflare official documentation for changing limits/APIs

