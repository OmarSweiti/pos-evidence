# Validation: 1.9.4, the business date at finalize

**Commit validated:** `20a1db8`, the pull request's head. It was checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:** nothing new. 1.8.3's finalize (#393) already copies a sale's business date from its cart's open shift. This pull request adds the step's three tests, in `commands::sale::business_date_tests`, and retires two of their names from the test catalog's `PLANNED` list.

The runtime claim is shown through a harness linked against this head's `terminal`, `pos-db`, `pos-domain` and `pos-hardware`:
- Each step is a new process that opens the register with `Register::open_with`, as `run` does, with its clock set to the instant the step names.
- It rings up a sale through the shipped commands, `sale_finalize` included.
- The receipt prints to 1.7.12's emulated printer through 1.7.10's network driver, both on `127.0.0.1` in the harness's process.

**How it ran:**
- **The suites (01):** run in the worktree:
  - the `Done when`, `cargo nextest run -p terminal business_date_`;
  - every `terminal` test;
  - the frontier check.

  Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–04):**
  - Built `--offline` against the worktree, with a fresh synthetic register file per scenario.
  - Each step's exit status is recorded beside its output.
  - In one scenario the store's zone is changed in the register's file, and the step says so.
  - The board builder refuses an unexpected exit, a failed check, the fixture's PIN or a local path.
- **The planted defects (05):**
  - 3 of them, run at this head before the pull request opened.
  - Each was run against the `Done when`'s selection, then restored with git.
  - The worktree was clean afterwards.
- **Builds ran without debug information** (`CARGO_PROFILE_DEV_DEBUG=0`).

**Data is synthetic throughout:**
- the reference chain's fixture merchant and its store, in Amman with a 04:00 day cutover;
- register `REG01`, with its shift open on 2026-11-05;
- cashier C-100, whose PIN is typed by the harness and printed nowhere;
- `SYN-` products.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal business_date_` | the three named tests pass | **pass**: 4 of 4: the three, and an existing clock test whose name the filter also matches (01) |
| 2 | Every `terminal` test, the frontier | all pass; reconciled | **pass**: 105 of 105; the frontier at 78 of 124 (01) |
| 3 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `20a1db8`: 980 passed, with 4 skipped (the existing `#[ignore]` ones). nextest flagged one existing pos-db test, `auth::tests::a_pin_hash_stored_under_argon2_0_5_still_verifies`, as leaky; it passed, and this pull request does not touch it. 79 web tests; every guard self-test; the test catalog reconciles (01) |
| 4 | A sale with the clock at 22:00 UTC on 5 November, 01:00 on the 6th in Amman, before the 04:00 cutover | the sale and its stock dated by the shift, 2026-11-05 | **pass**: completed at `2026-11-05T22:00:00.000Z`, business date 2026-11-05, its stock posted on 2026-11-05. The receipt prints both (02) |
| 5 | The store's zone set to Kiritimati, UTC+14, with the clock at 13:00 UTC on the 6th | still the shift's date | **pass**: business date 2026-11-05, its stock posted on 2026-11-05 (03) |
| 6 | The clock at 31 December | the instant the clock's, the business day the shift's | **pass**: completed at `2026-12-31T09:00:00.000Z`, business date 2026-11-05, its stock posted on 2026-11-05 (04) |
| 7 | 3 planted defects | each turns the test that names its case red | **pass**: 3 of 3 caught, each compiled (05) |

`protected-paths` is red by design. It names exactly one path, the byte-pinned `scripts/check-test-catalog.py`, whose two-name `PLANNED` retirement needs the owner's reading. Every other required check except `validation` is green at this head.

## Not validated here, and why

- **The operating system's zone was not changed.** Nothing in the shell, the domain or the database layer reads it, so changing it could not move a date. The store's zone, the one the register reads, is the one the scenario moves.
- **No screen calls the finalize yet.** That is 1.11.7's. The harness calls the same plain function the Tauri command wraps.
