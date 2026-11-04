# Validation: 1.9.5's domain and database half, the minimal shift lifecycle

**Commit validated:** `ac6a440`, the pull request's head, checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **`pos_domain::shift`**:
  - the business-date rule, `opening_business_date`;
  - the float counted by denomination, `count_float`;
  - `open_shift`, `Shift::close`, `Shift::till`.
- **`pos_db::repo::shift::ShiftRepository`**:
  - writes the opening and close facts, each after its envelope;
  - reads the open shift back;
  - refuses a second open shift, another user's close, a double close, a missing or different envelope, and a float in another currency.

No screen or command calls these yet. `shift_open`, `shift_current`, `shift_close` and the two screens are this step's deferred half, under its `Full-step status:` marker. The runtime is therefore shown through the domain and the repository themselves, on a synthetic register file.

**How it ran:**
- **The suites (01).**
  - The `Done when`'s domain and database parts: `cargo nextest run -p pos-domain shift::tests::` and `cargo nextest run -p pos-db --test shift_lifecycle`.
  - The property again at `PROPTEST_CASES=20000`.
  - `0005`'s own suite with its shift gates, every `pos-db` test, clippy, the whole workspace, the test catalog, and the frontier.
- **The harness (02, 03).**
  - It is linked against this head's `pos-db` and `pos-domain` and built `--offline` from the worktree's `Cargo.lock`.
  - Every step is its own process on one register file.
  - A shift is opened and closed the way its commands will do it: the domain decides, and the repository writes each fact with its audit row under one envelope.
  - A sale is made the way 1.8.3's finalize will make it.
- **The planted defects (04).** 19 defects, each applied to a scratch worktree at `ac6a440`, tested, and restored. No migration was mutated.

Data is synthetic throughout: an invented store in Amman, registers `REG01` and `REG02`, cashiers `FIX-1` and `C-2`, and one product, `SYN-OIL`.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | The `Done when`'s domain and database parts | all pass | **pass**: 7 of 7; 9 of 9 (01) |
| 2 | The property at 20,000 cases; `0005`'s suite; `pos-db`; clippy; the workspace; the catalog and the frontier | all pass | **pass**: 1 of 1; 14 of 14; 314 of 314; clippy clean; 870 passed, 4 skipped (the existing `#[ignore]` ones); both checks reconcile. nextest flagged the existing `auth::tests::a_new_pin_has_the_digits_its_account_needs` as leaky in this run. It passed, and this change doesn't touch it (01) |
| 3 | A shift opened at 01:30 Amman time on a trusted clock, with two twenty-dinar notes and a ten | business date the 25th, before the 04:00 cutover; float 50.000; an audit row saying the date was derived | **pass**: S-60 dated 2026-08-25, float 50.000 JOD; `shift.open` with `business_date_source` `derived` (02) |
| 4 | A second shift on the same register | refused by name, nothing written | **pass**: `ShiftAlreadyOpen { register: REG01, open: S-60 }` (02) |
| 5 | A supplied date on a trusted clock; no date on an untrusted one; a thirty-dinar note | each refused | **pass**: `SuppliedDateRefused`; `ConfirmedDateRequired`; `NotADenomination(30.000 JOD)` (02) |
| 6 | An untrusted clock with the operator's date | opened on that date, and audited as the operator's | **pass**: S-62 dated 2026-08-26, float 41.000. The audit row has reason "business date confirmed by the operator", `operator_confirmed` and `untrusted` (02) |
| 7 | A sale in S-60 dated the 25th; another dated the 26th | the first completes; the second is refused | **pass**: `REG01-000001` completed; the second refused by `0005`'s shift gate (03) |
| 8 | C-2 closes FIX-1's shift | refused by the domain, and by the repository asked directly | **pass**: `NotTheOpener`; `ShiftClosedByAnother` (03) |
| 9 | FIX-1 closes it, then tries again | closed with no approval, as a fact of its own; the second close refused | **pass**: a close event and a `shift.close` audit row; the opening fact unchanged; then `ShiftNotOpen(S-60)` (03) |
| 10 | A sale with no open shift | refused | **pass**: refused by `0005`'s shift gate; no sale written (03) |
| 11 | The next shift on REG01, at 08:00 on the 26th; the till, left in training in S-60; a sale | dated the 26th; the new till out of training; the sale completes | **pass**: S-63 dated 2026-08-26. S-60's till was left with `is_training = true`; S-63's till starts `false`. `REG01-000004` completed in S-63 (03) |
| 12 | 19 planted defects | each turns the test that names its rule red | **pass**: 19 of 19 caught. Every mutant compiled, and the worktree was restored with no changed file (04) |

`just pre-push` was run by `just pr` before the pull request was opened, and passed: lint, tests, the web build, guards and the secret scan.

## Not validated, and why

- **The commands and the screens.** `cargo nextest run -p terminal --test shift_lifecycle` and the `ShiftOpen` and `ShiftClose` `vitest` runs are the rest of the `Done when`. They need the register's database and the signed-in user in the shell's state, which no microstep's `Files:` line yet creates. The step's `Full-step status:` marker says how each command calls this half.
- **The shell clearing its own training flag at close.** That is the terminal half of E.54. Here the domain shows a new shift's till always starts out of training.
- **The required `protected-paths` check is red by design.** This pull request retires its `PLANNED` test name in the byte-pinned `scripts/check-test-catalog.py`, as every earlier retirement did.
