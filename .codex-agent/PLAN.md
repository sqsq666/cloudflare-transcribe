# PLAN.md - Autonomous Development Planning

## Architecture Baseline
- Production: Zero VPS, Cloudflare-only (Workers, R2, D1, Workflows, Queues, Containers, Workers AI)
- Local dev: Docker, FFmpeg for testing only (never production)
- Audio never enters Worker control body; browser -> R2 direct (with signed temp upload)
- Durable state in D1 (metadata) + R2 (audio/results)
- Ephemeral Container FS (/tmp)

## Ordered Phases
1. Domain contracts & state machines
2. API handlers & authorization
3. Adapters (D1, R2, AI, Container, Queue, Workflow)
4. Workflows & durable orchestration
5. Static client (web/)
6. Container image & FFmpeg handler
7. Tests, migrations, docs

## Dependencies
- None (initial planning)

## Acceptance Criteria
- All files in target layout
- Domain runtime-agnostic
- Cloudflare-only constraints satisfied

## Tasks (to be filled in implementation runs)
