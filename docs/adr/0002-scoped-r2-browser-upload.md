# ADR 0002: Direct browser uploads with scoped R2 authorization

- Status: Accepted, pending DEV capability verification
- Date: 2026-09-02
- Scope: Browser-to-storage data plane

## Decision

The Worker creates the job and issues short-lived authorization scoped to one
private R2 bucket and the job's `original/{job_id}/` prefix. The browser sends
audio directly to R2. Multipart upload is used for large files and records
upload ID, part numbers, and ETags so failed parts can be retried and a page
reload can resume. The Worker receives metadata-only completion confirmation
and verifies the finished object before processing.

Prefer R2 temporary credentials for a multi-operation multipart session; use
single-operation presigned URLs or per-part signatures only where the current
client/CORS capability test demonstrates a better fit. Parent R2 credentials
never reach the browser.

## Context

Worker request bodies are not a suitable path for hundreds of MB or 1GB+
uploads. Current R2 documentation supports resumable multipart objects up to
5 TiB, with up to 10,000 parts and a 5 MiB minimum for non-final parts.

## Consequences

Part size must be computed against the 10,000-part limit, CORS must be limited
to the application origin, and bearer credentials require short TTLs and
least-privilege paths/actions. Incomplete uploads need explicit abort/reaper
behavior plus lifecycle defense. Exact browser temporary-credential semantics
remain a DEV gate and must be documented before enabling production behavior.

