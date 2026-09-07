# Cloudflare-only Audio Transcription

This repository is building a browser-based audio transcription platform with
Cloudflare as the production runtime. It is designed for M4A (including Apple
Voice Memos), MP3, WAV, FLAC, OGG, WebM, and AAC recordings from short clips
through 1GB+ and multi-hour files.

The production architecture has zero VPS: the browser uploads directly to a
private R2 bucket, D1 stores structured job metadata, Workflows and Queues
coordinate durable processing, Containers run ephemeral FFmpeg/ffprobe work,
and Workers AI provides the standard Whisper transcription model. Source audio
is removed promptly after successful processing, with lifecycle cleanup as a
defense in depth.

This branch is currently at the architecture/planning checkpoint. Start with:

- [`PROJECT_SPEC.md`](PROJECT_SPEC.md) - product and non-negotiable requirements
- [`PLAN.md`](PLAN.md) - ordered implementation tasks and acceptance gates
- [`STATE.md`](STATE.md) - concise handoff for the next autonomous run
- [`docs/architecture.md`](docs/architecture.md) - data flow, contracts, and limits snapshot
- [`docs/adr/`](docs/adr/) - accepted architectural decisions and open capability gates

Local development is intentionally implemented before Cloudflare DEV resource
provisioning. See [`AGENTS.md`](AGENTS.md) for repository rules, privacy
requirements, and the expected validation workflow.
