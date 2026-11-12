# Validation: 1.7.14c, the print worker prints off the command path

**Commit validated:** `7332a29`, the pull request's head. It was checked out clean in a scratch worktree with its own build directories.

**What changes at runtime:**
- **`sale_finalize` kicks the drawer, hands the receipt to the print worker, and answers at once:** `sent_to_printer` with a printer bound, `queued` without one. It never waits on the device.
- **The print worker (`print_service`), started by `run`, prints off the command path.** It drains as it starts, on every finalize's nudge, and every five seconds. It holds the register's database only to lease a job and to record its attempt, never while the device is written to.
- **A receipt rendered for another profile is withheld,** never sent, and left for a person.

The runtime is shown through a harness linked against this head's `terminal`, `pos-db`, `pos-domain` and `pos-hardware`, built `--offline` in its own build directory.
- **Each scenario is one process.** It builds the shell suites' own shop file through `pos_db::open` and binds REG01 to 1.7.12's emulated printer under `emulator-80mm`.
- **It drives the shipped code:** it opens the till with `Register::open_with` and starts the worker with `print_service::start`, as `run` does, and sells through the shipped commands.

**How it ran:**
- **The suites (01):** run in the worktree. 1.7.14's `Done when` is run once in full and ten times more, because its worker test waits on a thread. Every `terminal`, `pos-db` and `pos-hardware` test runs with every feature, and clippy with `-D warnings`. Then `just pr`'s gate at this same head is quoted from its log.
  - On one of the ten repeats, nextest flagged `a_register_renders_for_its_bound_profile` as leaky. The test passed, and the other ten runs showed no leak. An existing `pos-db` auth test has shown the same intermittent flag on this machine (#428's gate).
- **The harness (02–03):** the board builder refuses any of the following: a non-zero exit, the synthetic PIN, the word `argon2`, a UUID, or a local path. It also asserts the key lines the steps below name. The finalize's time is shown rounded up to 100 ms, as information; nothing asserts on it.
- **The planted defects (04):** 7 of them, run at this head before `just pr` against 1.7.14's suite and the shell's own tests, each restored with git. The worktree was clean afterwards.

**Data is synthetic throughout:** the reference chain's fixture store with register REG01 and a shift open, ten bottles of oil, and the synthetic cashier C-100.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | 1.7.14's `Done when`, once and ten times more | 5 of 5 each run, the step's four named tests among them | **pass**: 11 runs, all 5 of 5; one run flagged one test leaky, still passing (01) |
| 2 | Every `terminal`, `pos-db` and `pos-hardware` test; clippy | all pass; clean | **pass**: 621 of 621, 2 skipped (existing); clippy exit 0 (01) |
| 3 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `7332a29`: 1043 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test (01) |
| 4 | A sale with the worker not yet started | the finalize answers `sent_to_printer`, the drawer kicked; the job still queued; nothing at the device | **pass**: `sent_to_printer`, `opened`; `queued`, no attempt; the emulator has received 0 jobs (02) |
| 5 | The worker started against a printer answering slowly | while it holds the job, commands needing the database answer, and the job is still printing; then it prints | **pass**: `printing`; `stock_on_hand` (9 on hand) and `shift_current` answer; still `printing`; then `printed`, cut; health raises nothing (02) |
| 6 | A receipt kept unbound, then the register bound and the worker started | the worker's first pass withholds it, never sending a byte | **pass**: `failed:profile_mismatch` on the simulator's receipt (03) |
| 7 | The next sale | its finalize nudges the worker, which prints it; only that job reaches the device; the withheld one counted as waiting | **pass**: `printed`; the emulator received 1 job; `receipts_waiting` 1 (03) |
| 8 | 7 planted defects | each turns the test that names its rule red | **pass**: 7 of 7 caught, each compiled (04) |

## Not validated here, and why

- **A refusal retried when due, with the real clock.** The harness's register runs on the fixture's fixed clock, so a retry thirty seconds later never comes due. `a_queued_artifact_prints_unchanged_once_a_printer_returns` and `hardware_failure_after_commit_leaves_sale_complete` cover it with a movable clock.
- **No screen.** No screen reads the finalize's answer yet; the status strip that will carry print failures is 1.11.x's.
