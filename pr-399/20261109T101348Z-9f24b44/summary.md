# Validation: 1.8.4's recovery half, a sale a restart left taken up again

**Commit validated:** `9f24b44`, the pull request's head. It was checked out clean in a scratch worktree with its own build directories.

**What changes at runtime:**
- **Start-up completes nothing.** It still finds each journalled sale and leaves it where it is.
- **A new command, `sale_resume_in_flight { sale_id }`,** puts one on the till, as the domain's new `resume_in_flight` decides from the journal alone:
  - nothing collected: the checkout is abandoned, `checkout.abandon` is audited, and its basket comes back;
  - tenders taken: the tender screen, with every one;
  - a finalize begun: `sale_finalize` replays it under the journal's own key and version.
- **Another cashier may take one over,** audited as `sale.resume`. Every row written before the sale completes names its basket, `parked_cart`, under the sale's id (#397).

The runtime is shown through a harness linked against this head's `terminal`, `pos-db`, `pos-domain` and `pos-hardware`.
- **Each step is a new process.** It opens the register with `Register::open_with` on a file built by `pos_db::open`, as `run` opens it, so start-up recovery runs every time.
- **A power cut is `abort()`.** No destructor runs and nothing more commits. `die` aborts between commands. `die-in:sale_line` aborts inside the finalize's one transaction, at its first `sale_line` row, through a SQLite update hook.
- **It drives the shipped commands:** sign-in, the cart, the tenders, `tender_back_to_cart`, `sale_in_flight_state`, `sale_resume_in_flight` and `sale_finalize`.
- **Idempotency keys are never printed.** Each is named by the order it was first seen in, kept in a file beside the register, so one can be compared across processes.
- **The receipt prints** to 1.7.12's emulated printer through 1.7.10's network driver, on the 80 mm profile, both started in the harness's process on `127.0.0.1`.

**How it ran:**
- **The suites (01):** run in the worktree:
  - 1.8.4's `Done when`, `cargo nextest run -p terminal recovery::`;
  - the domain's `resume_in_flight` tests, its property at 4,096 cases;
  - 1.8.3's test that a restart leaves a journalled sale alone;
  - every `pos-domain` and `terminal` test;
  - clippy over both crates with `-D warnings`.

  Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–06):**
  - Built `--offline` against the worktree, with a fresh synthetic register file per scenario.
  - Each step's exit status is recorded beside its output: 0, or 134 for a step meant to die.
  - The board builder refuses an unexpected exit, a failed check, the fixture's PIN, a UUID or a local path. It also requires every scenario to complete its sale.
- **The planted defects (07):**
  - 14 of them, run at this head before the pull request's second gate.
  - Each was run against the tests its rule belongs to, then restored with git.
  - The worktree was clean afterwards.
- **Builds ran without debug information** (`CARGO_PROFILE_DEV_DEBUG=0`).

**Data is synthetic throughout:**
- the reference chain's fixture merchant and its store;
- register `REG01`, with its shift open on 2026-11-05;
- cashiers C-100 and C-200, whose PIN is typed by the harness and printed nowhere;
- `SYN-` products.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal recovery::` | the five named tests and the command's four pass | **pass**: 9 of 9 (01) |
| 2 | The domain's three `resume_in_flight` tests; 1.8.3's `a_sale_left_mid_payment_stays_journalled_for_recovery` | all pass | **pass**: 3 of 3, and 1 of 1 (01) |
| 3 | Every `pos-domain` and `terminal` test; clippy | all pass; clean | **pass**: 400 of 400 and 116 of 116; clippy exit 0 (01). nextest reported two passing tests leaky (their output handles closed after its 100 ms window); the gate's run reports none |
| 4 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `9f24b44`: 994 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog and the frontier reconcile (01) |
| 5 | C-100 begins paying for oil and 101 g of cheese; the process dies; restart, take it up, pay 2.110, finalize | start-up parks nothing; it comes back as its basket, the abandon audited on the basket; the sale completes and prints | **pass**: start-up left 1 sale journalled and set 0 baskets aside; resumed at `building`, 2 lines, 2.108; `checkout.abandon` on `parked_cart`, in a commit of its own; a new key journalled for the new payment; `REG01-000001`, receipt `Printed`, drawer opened; the sale's commit whole, 13 members, 13 delivery rows (02) |
| 6 | C-100 takes 1.000; the process dies; restart, take it up, pay 1.110, finalize | the tender screen with its tender, owing 1.108, under the journal's own key and version | **pass**: resumed at `tendering`, `[cash 1.000]`, owed 1.108, cash asked 1.110; the till under key #1 at version 1; completes `REG01-000001` with both tenders, receipt `Printed` (03) |
| 7 | C-100 settles with 1.000 and 5.000; the power is cut inside the finalize; again inside the replay; then the replay completes | nothing of the cut attempts left; the journal not advanced by the replay; the sale completed once | **pass**: after the first cut the journal was `finalizing` at version 3, with no sale, no sale stock event, receipt number 1 unused. After the second cut it was unchanged, same key, version 3. The replay completed `REG01-000001`, change 3.890; stock posted once (oil 9.000, cheese 0.399); one commit, 15 members, 15 delivery rows (04) |
| 8 | C-100 takes 1.000; the process dies; C-200 signs in and takes it over, with the refusals first | an unknown sale and a till already paying refused; C-200's own basket set aside; the takeover audited on the basket; the sale stays C-100's | **pass**: `sale_not_in_flight`; `sale_being_paid`; after going back to their basket, resumed at `tendering`; C-200's basket in the parked list; `sale.resume` on `parked_cart` in a commit of its own; `REG01-000001` is C-100's, its finalize C-200's, receipt `Printed` (05) |
| 9 | C-100 begins paying; the process dies; C-200 takes it up, pays 2.110, finalizes | the basket becomes C-200's; both rows name the basket; the sale still completes | **pass**: resumed at `building`; `checkout.abandon` and `sale.resume` both on `parked_cart`, each in a commit of its own; `REG01-000001` is C-200's, receipt `Printed`; the sale's commit whole, 13 members, 13 delivery rows (06) |
| 10 | 14 planted defects | each turns the test that names its rule red | **pass**: 14 of 14 caught, each compiled (07) |

## Two things that went wrong on the way, and were fixed

- **The first `just pr`, at `8e9ed75`, stopped in its test stage.** 1.4.10's `every_privileged_transition_returns_an_audit_intent` requires every public function of the cart module to be listed with its audit duty, and `resume_in_flight` was not. `9f24b44` lists it as unaudited, as `complete` is, because the shell writes the abandon's and the takeover's rows. The workspace then passed 994 of 994 with `--no-fail-fast`, and the 14 planted defects were run again at `9f24b44`.
- **The defect runner's first pass, at `8e9ed75`, misread three of them.** Each run exited with failures, but no failing test was read from its output, and no log was kept to say why. A fourth was missed because its test wasn't in the run's selection. The runner now saves every mutant's output and reads only single-line status lines. Two full passes since, at `8e9ed75` and at `9f24b44`, caught all 14.

## Not validated here, and why

- **A card operation outstanding at a restart.** No Phase-1 tender has one, and the journal cannot store one. The two tests moved to 2.1.3, as the owner decided. The domain's `QueryCardFirst` is proven by its tests; M01 turns them red.
- **The screen.** The recovery screen and its `sale://recovered` event are 1.11's. The harness calls the same plain functions the Tauri commands wrap.
- **A journalled sale whose shift has closed.** It cannot complete, because the finalize refuses `shift_not_open`. Refusing to close a shift with a sale in flight is the shift-close step's.
- **The receipt's time is printed in UTC**, as on every receipt today. That is an open question for the owner, outside this step.
