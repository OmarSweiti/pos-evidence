# Validation: 1.8.1b, a half-migrated or altered database is refused, by name

**Commit validated:** `5238454`, the pull request's head, checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- `pos_db::open` now holds a file's schema to the one its recorded `user_version` creates, before the first migration and after each. It refuses any difference as `SchemaShapeMismatch`, with the file untouched.
- The forensic `verify-audit` reads such a file anyway and names the difference. The register has no screen yet, so the register's side is shown through `pos_db::open` itself, which is what the app will call.

**How it ran:**
- **The suites.** The `Done when`, every `pos-db` test, clippy, and the whole workspace (01).
- **The harness.** A harness linked against this head's `pos-db`, built `--offline` from the worktree's `Cargo.lock`. It builds synthetic register files the way the runner does, interrupts each step three ways, and opens each one, recording the version and object count before and after (02, 03).
- **The verifier.** The real `verify-audit` binary from this head on a synthetic register and three copies of it, with its exit codes (04).
- **The planted defects.** 13, each applied to the worktree, tested and restored (05).

Data is synthetic throughout: invented ids, an invented register, an invented key that no image shows.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db --test migration_refusal` | 4 tests pass | **pass**: 4 of 4 (01) |
| 2 | `pos-db` with all features, clippy `-D warnings`, the workspace | all pass | **pass**: 267 of 267; clippy clean; 800 passed, 4 skipped, the existing `#[ignore]` ones (01) |
| 3 | Steps 0001 to 0007, each interrupted three ways: the step without its version, the version without its step, half its statements | each refused by name, naming what differs, and the file unchanged | **pass**: 21 refusals, each file's version and object count unchanged (02, 03) |
| 4 | Step 0008, which changes rows alone, interrupted the same three ways | it opens, because no schema shows it, as the PR states | **as stated**: all three open. Two reran it (version 7 to 8, nothing else changes); the version without its rows opens unchanged, the limit the PR records (03) |
| 5 | A version one past this build's chain | refused as a newer build's, untouched | **pass**: `SchemaTooNew`, version 9 kept (03) |
| 6 | A file stopped cleanly at 0004 | opens and migrates forward through the check | **pass**: from 4 to 8 (03) |
| 7 | The append-only audit trigger redefined under its own name to do nothing | refused as altered | **pass**: `altered: trigger audit_log_no_update` (03) |
| 8 | A register file and three copies, opened by `pos_db::open` | the original and the careful copy (trigger restored) open; the copies with a dropped trigger are refused, naming it | **pass** (04) |
| 9 | `verify-audit` on each | the original verified (exit 0). A dropped trigger is named on the schema line and counted (exit 1) while the chain is still walked. The edited row is named either way | **pass**: exits 0, 1, 1, 1. `missing trigger audit_log_no_update` on both dropped copies; `BROKEN at seq 3` on both edited ones (04) |
| 10 | 13 planted defects | 12 turn the test that names their rule red; the 13th survives, as predicted | **pass**: 12 of 12 caught. M13 (no check after each step) survived. Every mutant compiled, and the worktree was restored with no changed file (05) |

`just pre-push` was run by `just pr` before the pull request was opened, and passed: lint, tests, the web build, guards and the secret scan.

## Not validated, and why

- **A register app refusing on screen.** No screen opens the database yet. The named recovery screen for this state is 1.8.7's. The refusal is shown here at `pos_db::open`, the call the app makes.
- **Step 0008 half-applied.** No schema shows a step that changes rows alone, so the check can't see one. The pull request states it, and the test lists 0008 by number. Closing it needs a persisted record of each migration, and so a new migration, which is left for the owner to decide.
