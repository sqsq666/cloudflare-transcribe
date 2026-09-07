# ADR 0001: Cloudflare-only durable architecture

- Status: Accepted
- Date: 2026-09-02
- Scope: Production runtime and durable data

## Decision

Run the production control plane on Cloudflare Workers with Workers Static
Assets. Store durable objects and result files in private R2, structured state
in D1, job orchestration in Workflows, independent chunk work in Queues, and
FFmpeg/ffprobe execution in Cloudflare Containers. Keep Container filesystems
ephemeral scratch only. The development workstation may run Docker and local
FFmpeg, but it is never a production dependency.

## Context

The product must handle multi-hour and 1GB+ recordings while enforcing zero
VPS and zero permanent audio storage. A traditional server or persistent
container volume would violate the architecture rule and would make restart
and privacy behavior harder to prove.

## Consequences

Large artifacts are referenced by R2 keys instead of being carried in Workflow
state, Queue messages, or D1. Every processing step must be restart-safe and
idempotent. Streaming and bounded scratch usage are required, and current
Cloudflare limits must be rechecked at each integration gate.

## Rejected alternatives

An always-on VPS, Docker Compose host, Nginx upload proxy, relational server,
Redis queue, or persistent Container volume would violate the zero-VPS and
Cloudflare-only requirements and is not an acceptable fallback.

