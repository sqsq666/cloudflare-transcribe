# Repository operating rules

This repository implements the Cloudflare-only audio transcription platform
described in `PROJECT_SPEC.md`. These rules apply to every contributor and
autonomous development run.

## Non-negotiable architecture

- Production has zero VPS and no persistent traditional server.
- Production runtime is limited to Cloudflare Workers, Workers Static Assets,
  R2, D1, Workflows, Queues, Containers, Workers AI, and other explicitly
  approved Cloudflare services.
- The development workstation, Docker, FFmpeg, and local emulators are never
  production dependencies.
- Large audio bytes must go browser -> private R2 directly. A Worker control
  request may create a job, issue narrowly scoped temporary upload access, and
  confirm metadata, but must not proxy an upload body.
- R2 is the only durable store for audio and result artifacts. D1 stores
  structured metadata and state, never large audio or transcript blobs.
- Container filesystems are scratch space only (`/tmp`). A restart must not
  lose durable state or be required for correctness.
- Queue messages contain identifiers and bounded metadata, never audio bytes.
- Permanent credentials, secrets, and user content must never enter Git.

## Development workflow

1. Work on `dev`. Fetch `origin/dev` before starting and incorporate remote
   changes safely; never force-push or rewrite shared history.
2. Read `PROJECT_SPEC.md`, this file, `PLAN.md`, `STATE.md`, and the current
   worktree status before acting.
3. Select the next incomplete task in `PLAN.md`. Keep one atomic milestone per
   autonomous run when practical.
4. Prefer local implementations and deterministic fakes until the plan's DEV
   gate is reached. Do not create or modify production Cloudflare resources.
5. For changing Cloudflare behavior, consult current official documentation
   and record the URL, date, and relevant constraint in `docs/architecture.md`
   or an ADR.
6. Before editing, inspect overlapping worktree changes and preserve them.
7. After implementation, run the task's stated tests, diagnose failures, and
   rerun validation. Do not knowingly leave a broken branch.
8. Update `PLAN.md` and `STATE.md` in the same milestone commit. Keep state
   concise enough for a fresh agent to resume.
9. Commit a meaningful conventional-style message and push `dev` to
   `origin/dev`. Never deploy production from this loop.

## Target layout

The implementation should converge on this layout unless an ADR records a
change:

```text
src/
  domain/       pure contracts, state machines, planner, merge, exporters
  api/          Worker HTTP handlers and authorization
  adapters/     D1, R2, AI, Container, Queue, Workflow implementations
  workflows/    durable job orchestration
web/            Vite/React static client
container/      FFmpeg/ffprobe image and command handler
migrations/     ordered D1 SQL migrations
tests/          unit, integration, security, and browser fixtures
docs/           architecture, ADRs, runbooks, and DEV verification
```

Keep domain code runtime-agnostic and make external bindings injectable. Do
not hide network, storage, or clock access in pure functions.

## Data and privacy rules

- Generate unpredictable job and object identifiers with Web Crypto.
- Enforce ownership on every job, upload, transcript, export, and deletion
  operation; authorization is checked server-side, not just in the UI.
- Validate declared extension/MIME, size, and ffprobe-detected media before
  processing. Treat client metadata as untrusted.
- Persist only the minimum metadata needed for progress, recovery, billing
  accounting, and auditability. Redact audio text and credentials from logs.
- On success, persist canonical results, delete the source and successful temp
  objects promptly, and leave no local container copy. Failed source cleanup
  targets 24 hours; result cleanup targets 7 days, with R2 lifecycle rules as
  defense in depth.
- "Delete all data now" must remove all content-bearing R2 objects and D1
  records. A tombstone is allowed only when it contains no user content.

## Secrets and generated files

Use `.env.example` for names only. Local values belong in ignored `.env`,
`.dev.vars`, or Wrangler secret storage. Never print secrets in diagnostics,
tests, commits, pull requests, or logs. Run a secret scan before pushing.
Do not commit build output, Wrangler state, local databases, recordings,
transcripts, private fixtures, or credentials.

## Quality bar

Completed milestones require TypeScript typecheck, lint, formatting, and the
focused unit/integration tests to pass. Add tests with every stateful or
security-sensitive behavior. Important tests include:

- legal and illegal job/chunk transitions, transactional races, and retry
  idempotency;
- multipart retry/resume/abort and proof that audio never enters a Worker
  control body;
- all supported formats, ffprobe metadata, bounded FFmpeg scratch usage, and
  adaptive chunk boundaries;
- timestamp offset, overlap deduplication, speaker reconciliation, and export
  golden files;
- ownership, signed access, expiry, deletion, rate limiting, and privacy
  guarantees;
- Playwright desktop/mobile upload, progress, transcript, search, rename,
  export, and delete flows once the client exists.

Use deterministic clocks, IDs, and provider fakes in unit tests. Use real
Cloudflare DEV resources only in tests explicitly marked as DEV integration.

## Documentation and handoff

`PLAN.md` is the ordered source of work and `STATE.md` is the resume point.
Record material architecture changes as numbered ADRs. Mark a task complete
only when its acceptance criteria and tests are met. Use `.codex-agent/BLOCKED`
only when external credentials or an unavoidable human decision is genuinely
required; transient service failures are retryable and are not blockers.

