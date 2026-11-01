# Validation: microstep 1.8.1, transaction discipline held by a test

**Commit validated:** `eae9875`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- **A test-only change.** No production code changed apart from a doc comment in `repo/mod.rs`, so there is no screen to drive. The evidence is the guard itself: what it finds, what it refuses, and what it catches when a violation is planted.
- **A second reader**, written separately in Python for this evidence and not in the repository, read the pull request's own `crates/pos-db/src` and listed every shipped function that writes a row or controls a transaction. Its count is set beside the test's.
- **Planted changes:** each one changed one place in `pos-db`'s shipped source in the worktree, ran `repository_transaction_api`, and was restored. The last is a control that no test may flag. The run ends with "restored: 0 changed files", and the second reader ran after it, on the clean worktree.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db --test repository_transaction_api` (this step's `Done when`) | the four tests pass, the compile-fail lane among them | **pass**: 4/4 (01) |
| 2 | The `pos-db` crate, the workspace | no failure | **pass**: 249/249; 765/765 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | A second reader of the shipped source | every function that writes a row takes the caller's transaction; only the migration runner controls one | **pass**: 15 writers, 0 without; `migrate` alone (02) |
| 4 | The compile-fail fixture | an audit append and a commit manifest handed a `&Connection` are refused | **pass**: the committed golden shows both `E0308` refusals (02) |
| 5 | Seven violations planted in shipped source | each caught, and named by file, line and function | **pass** (03) |
| 6 | One control: a comment mentioning `INSERT INTO` in a function that only reads | no test fails | **pass**: 4/4 (03) |

## Notes

- **The rule was already kept.** Every repository write built so far took a `&Transaction`, so this step adds a guard and fixes no violation. The second reader and the test agree on the same fifteen writers.
- **The scan reads Rust carefully, and two real cases showed why it must.** A `'"'` character literal in `search.rs`, and the field name `"commit_index"` in `outbox.rs`, each fooled the first version of the scanner while it was being written. `the_scan_sees_a_write_and_a_transaction_in_any_spelling` now holds the reader to every kind of literal and to whole-word keywords.
- **The "save commits on its own" defect is caught by the scan alone.** Its commit runs before the save's insert, so the rollback test still sees that insert undone. Catching it is the scan's job.

## Not validated, and why

- **Repositories not yet written** (1.8.2 onward): the test holds them to the same rule as they land.
