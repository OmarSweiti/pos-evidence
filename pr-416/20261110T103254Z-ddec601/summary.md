# Validation: D6, a shift is never dated before the register's last (1.9.5)

**Commit validated:** `ddec601`, the pull request's head. It was checked out clean in a scratch worktree with its own build directories.

**What changes at runtime:** `shift_open` refuses an opening dated earlier than the register's last shift.
- **The refusal** is `business_date_before_last_shift`, with the detail "the register's last shift is on a later day", and nothing is written.
- **The repository holds the rule** (`DbError::ShiftDateBeforePrevious`), whoever calls it.
- **The same day again is allowed,** for a second shift that day, and another register's days are its own.

This is the owner's decision of 10 November 2026.

The runtime is shown through a harness linked against this head's `terminal`, `pos-db`, `pos-domain` and `pos-hardware`.
- **Each step is a new process** on one register file built by `pos_db::open`, opened with `Register::open_with` as `run` opens it, at the clock it names.
- **It drives the shipped commands:** sign-in, `shift_open`, `shift_current` and `shift_close`, through the plain functions the Tauri commands wrap.
- **The register trusts no clock** (`Untrusted`), so every opening is dated by the operator, as the shipped command asks.

**How it ran:**
- **The suites (01):** run in the worktree:
  - both shift suites, with the rule's two new tests;
  - every `terminal` and `pos-db` test;
  - clippy over both crates with `-D warnings`.

  Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02):**
  - Built `--offline` against the worktree, in its own build directory.
  - The board builder refuses any of the following: a non-zero exit, an action whose outcome differs from the one written, a failed check, the synthetic PIN, a UUID, or a local path.
- **The planted defects (03):** 6 of them, run at this head before `just pr`.
  - Each was run against the suite its rule belongs to, then restored with git.
  - The worktree was clean afterwards.
- **Builds ran without debug information** (`CARGO_PROFILE_DEV_DEBUG=0`).

**Data is synthetic throughout:** the reference chain's fixture store, with register `REG01` and no shift ever opened, and cashier C-100 with one synthetic PIN.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | The shift suites | both pass, with the rule's two tests | **pass**: 10 of 10 and 11 of 11, `a_shift_is_never_dated_before_the_registers_last` and `an_opening_dated_before_the_registers_last_shift_is_refused` among them (01) |
| 2 | Every `terminal` and `pos-db` test; clippy | all pass; clean | **pass**: 494 of 494; clippy exit 0 (01) |
| 3 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `ddec601`: 1029 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog and the frontier reconcile (01) |
| 4 | Monday: a shift dated 2026-11-09 opens and closes; an evening shift on the same day opens and closes | both open; the same day again is a second shift that day | **pass**: shift #1 and shift #2, each dated 2026-11-09 (02) |
| 5 | Tuesday: an opening typed as Sunday, 2026-11-08, and one typed as last week, 2026-11-02 | each refused by name, nothing written, no shift open | **pass**: `business_date_before_last_shift` twice, with its detail; no shift open (02) |
| 6 | Tuesday, typed right, 2026-11-10 | it opens | **pass**: shift #3, dated 2026-11-10; the facts hold three shifts and their five audit rows, none from the refused attempts (02) |
| 7 | 6 planted defects | each turns the test that names its rule red | **pass**: 6 of 6 caught, each compiled (03) |

## Not validated here, and why

- **What the rule cannot catch.** Typing Monday's own date on Tuesday is allowed, because the date alone cannot tell it from Monday's evening shift. A trusted clock closes it by deriving the date, and the shift screen could ask the operator to confirm when the typed date equals the last shift's. The PR's "Read before approving" names both.
- **The shift screens.** They are 1.9.5's, waiting for 1.11.3b.
