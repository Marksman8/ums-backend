# Backlog

Ideas and follow-ups that are out of scope for the current phase. Nothing here is built without review.

| # | Item | Raised in | Notes |
| --- | --- | --- | --- |
| 1 | Allow DDL in the Dev console | Owner request (pre-P0) | Conflicts with master §2 DEV "no DDL". Needs owner decision + ADR before P6. |
| 2 | 2026-10-10 — P1: `ErrorReporter` writes uncaught errors to `app_error_log` via the audit-writer pattern (bounded queue, best-effort), replacing the no-op `UncaughtErrorSink`. | STOP C review | `UncaughtErrorSink` stays a no-op in P0. |
| 3 | 2026-10-10 — P1: upgrade `actions/checkout`, `actions/setup-java` and `actions/upload-artifact` in `.github/workflows/ci.yml` to their Node 24 major versions. | STOP D review | Currently pinned to `@v4`. |
| 4 | 2026-10-10 — P1: pin `runs-on` to `ubuntu-24.04` instead of `ubuntu-latest` (ubuntu-latest moves to Ubuntu 26 on 2026-10-19). | STOP D review | Applies to the `fast`, `verify` and `nightly` jobs. |
| 5 | 2026-10-10 — P1: decide whether the `Marksman8/ums-backend` repository should be private. | STOP D review | Currently public. |
