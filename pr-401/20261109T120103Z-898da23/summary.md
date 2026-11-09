# Validation: 1.9.5's shift commands, a shift opened, sold in, closed and opened again

**Commit validated:** `898da23`, the pull request's head. It was checked out clean in a scratch worktree with its own build directories.

**What changes at runtime:** three new commands, each a thin call into the domain and database half #372 delivered.
- **`shift_open { float_by_denomination, business_date? }`** counts the float in the dinar's own pieces, and decides the business date once. The register trusts no clock yet, so the date is the operator's, required and audited. The opening travels with its `shift.open` row in one envelope.
- **`shift_close { shift_id }`** is the opener's, with no approval. A sale mid-payment in the shift refuses it. The till's basket is set aside, as a sign-out sets it aside. The close is a fact of its own, travelling with its `shift.close` row in an envelope it names.
- **`shift_current`** answers without anyone signed in.

The runtime is shown through a harness linked against this head's `terminal`, `pos-db`, `pos-domain` and `pos-hardware`.
- **Each step is a new process.** It opens the register with `Register::open_with` on a file built by `pos_db::open`, as `run` opens it, with the clock the step names.
- **Every opening uses the clock's confidence as the shipped `shift_open` reads it,** from `Register::clock_confidence`: `Untrusted`.
- **A power cut is `abort()`.** No destructor runs and nothing more commits.
- **It drives the shipped commands' plain functions:** sign-in and sign-out, the shift commands, the cart, the tenders, `sale_in_flight_state`, `sale_resume_in_flight`, `cart_list_parked`, `cart_resume` and `sale_finalize`.
- **No id is printed.** Each shift and basket is named by the order it was first seen in.
- **Each receipt prints** to 1.7.12's emulated printer through 1.7.10's network driver, on the 80 mm profile, both started in the harness's process on `127.0.0.1`.

**How it ran:**
- **The suites (01):** run in the worktree:
  - the commands' ten tests, `cargo nextest run -p terminal --test shift_lifecycle`;
  - 1.9.5's domain and database halves;
  - the store's trading day;
  - every `terminal` and `pos-db` test;
  - clippy over both crates with `-D warnings`.

  Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–06):**
  - Built `--offline` against the worktree, with a fresh synthetic register file per scenario and no shift open until one is opened.
  - Each step's exit status is recorded beside its output: 0, or 134 for a step meant to die.
  - The board builder refuses an unexpected exit, an action whose outcome differs from the one written, a failed check, the fixture's PIN, a UUID or a local path. It also requires each scenario's sales to complete.
- **The planted defects (07):** 16 of them, run at this head before `just pr`.
  - Each was run against the suite its rule belongs to, then restored with git.
  - The worktree was clean afterwards.
- **Builds ran without debug information** (`CARGO_PROFILE_DEV_DEBUG=0`).

**Data is synthetic throughout:**
- the reference chain's fixture merchant and its store, with its day turning at 04:00 in `Asia/Amman`;
- register `REG01`;
- cashiers C-100 and C-200 and shift lead L-100, whose PIN is typed by the harness and printed nowhere;
- the `SYN-OIL-1L` product.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal --test shift_lifecycle` | the ten tests pass, the first the `Done when` through the shell | **pass**: 10 of 10 (01) |
| 2 | 1.9.5's domain and database halves; the store's trading day | all pass | **pass**: 7 of 7, 9 of 9, and 5 of 5 (01) |
| 3 | Every `terminal` and `pos-db` test; clippy | all pass; clean | **pass**: 472 of 472; clippy exit 0 (01) |
| 4 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `898da23`: 1006 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog and the frontier reconcile (01) |
| 5 | C-100 signs in on a register with no shift; opens a basket; opens a shift with no date, with a 300-fils piece, with `05/11/2026`, then confirmed for 2026-11-05; again; signs out and reads the shift | each refused by name; the shift opens on the confirmed date with the counted float; a second refused; the shift read with no one signed in | **pass**: `shift_not_open`, `business_date_required`, `float_not_a_denomination`, `business_date_invalid`; shift #1 opened, business date 2026-11-05, float 27.500 JOD; `shift_already_open`; read after sign-out (02) |
| 6 | C-100 sells; at 18:00 C-200, L-100, then C-100 with another shift's id, try to close; C-100 closes it | the sale completes in the shift; the close refused for all but its opener; the opening untouched; each fact in one ready envelope with its audit row | **pass**: `REG01-000001`, business date 2026-11-05, in shift #1, receipt `Printed`; `shift_not_yours` twice, `shift_not_current`; closed, no basket parked; then `shift_not_open` for a basket and a second close; opening state `closed`; envelopes `[shift, audit_log (shift.open)]` and `[shift_close_event, audit_log (shift.close)]`, 2 delivery rows each; the close names its envelope; the opening audited `operator_confirmed`, clock `untrusted` (03) |
| 7 | C-100 takes 1.000 toward a sale and tries to close; the process dies; after the restart, again; then resumes, pays and closes | the close refused before and after the crash; once the sale completes, the shift closes | **pass**: `sale_in_flight` twice, the second for a sale in flight on no till; resumed, paid 0.250, `REG01-000001` printed; closed; nothing in flight (04) |
| 8 | C-100 closes with a basket on the till; the next shift of the day resumes and sells it; closes with a basket begun for payment; the third shift closes with an empty basket | the basket parked and the till empty; resumed and sold; the unpaid checkout given back and parked; the empty basket dropped; each close's row names what it parked | **pass**: basket #1 parked, till empty; listed, resumed and sold as `REG01-000001` in shift #2; basket #2 parked, nothing in flight, till empty; shift #3 closed with none parked; audit rows `parked_cart` basket #1, basket #2, null (05) |
| 9 | On 6 November C-100 opens the next shift on the same register, confirmed for 2026-11-06, and sells | a new shift on the same register; each sale dated by its own shift; receipt numbers run on | **pass**: shift #2, business date 2026-11-06; `REG01-000002` printed; sales dated 2026-11-05 in shift #1 and 2026-11-06 in shift #2; both openings and the close each in their envelope with their audit row (06) |
| 10 | 16 planted defects | each turns the test that names its rule red | **pass**: 16 of 16 caught, each compiled (07) |

## A required check that is red, and why

**`supply-chain` failed** on RUSTSEC-2026-0459, a SQLite advisory published today. `cargo deny` flags `libsqlite3-sys` 0.37.0, which every branch locks; no change in this pull request touches it. The advisory's patched `libsqlite3-sys` still builds SQLCipher on an affected SQLite. The owner is asked how to proceed, outside this pull request.

## Not validated here, and why

- **The two screens.** `ShiftOpen.tsx` and `ShiftClose.tsx` wait for 1.11.3b's design direction, so the harness calls the plain functions the Tauri commands wrap.
- **A trusted clock's derived date at runtime.** The register trusts no clock yet, so no shipped path derives a date. The derivation from the store's zone and cutover is shown by `a_trusted_clock_dates_the_shift_by_the_stores_own_day` (01), and M02 and M03 turn it red.
- **The receipt's time is printed in UTC**, as on every receipt today. That is an open question for the owner, outside this step.
