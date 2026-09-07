# ADR 0003: Cloudflare-only diarization capability gate

- Status: Accepted, capability unresolved
- Date: 2026-09-02
- Scope: Speaker Detection mode

## Decision

Keep speaker entities, chunk-local speaker labels, reconciliation contracts,
and UI rename/filter behavior in the V1 architecture. Enable actual speaker
diarization only after a current Cloudflare-hosted model or service is
verified for the required Chinese and mixed-language cases. Until then,
Speaker Detection is disabled or clearly reported as unavailable. No
third-party or self-hosted production provider may be introduced silently.

## Context

The current official Workers AI Whisper Turbo model page documents batch audio
transcription and timestamped segments, but does not document a diarization
input/output contract. Chunk-local speaker IDs also cannot be assumed stable
across independently processed chunks.

## Consequences

The provider interface must represent optional speaker labels and uncertainty,
and merge tests must exercise reconciliation without inventing identities. A
capability spike at the DEV gate records model availability, language quality,
limits, and evidence. If no qualifying Cloudflare capability exists, the
limitation is documented and V1 proceeds without pretending that diarization
works.

