# Validation: 1.7.6b, no printer, and selling never stops

**Commit validated:** `198f25a`, the pull request's head. It was checked out clean in a scratch worktree with its own build directories.

**What changes at runtime:** a new command, `health_status () → HealthSnapshot`, which the lock screen reads before anyone signs in. It carries three things:
- **the printer as read now,** the reading Pay takes;
- **two counts:** receipts waiting for a printer, and prints only a person can close;
- **an alarm for each,** never a modal.

Every receipt was already kept in the sale's own transaction (1.8.3). It prints, byte for byte as stored, with the next sale once a printer can take it.

The runtime is shown through a harness linked against this head's `terminal`, `pos-db`, `pos-domain` and `pos-hardware`.
- **Each step is a new process** on one register file, the shell suites' own shop, opened with `Register::open_with` as `run` opens it, at the clock it names.
- **It drives the shipped commands:** sign-in, the cart, tender, the finalize and `health_status`, through the plain functions the Tauri commands wrap.
- **The printer, where there is one, is the real driver against the emulator:** 1.7.10's `NetworkPrinter` on the 80 mm profile over TCP, talking to 1.7.12's emulated printer, started in the same process on 127.0.0.1.
- **Every page is checked against storage:** after a run with a printer, each page the emulator received is held to the receipts the register stored, byte for byte.

**How it ran:**
- **The suites (01):** run in the worktree:
  - 1.7.6b's `Done when`: `cargo nextest run -p terminal print_queue::`;
  - the IPC contract;
  - every `terminal` and `pos-hardware` test;
  - clippy over both crates with `-D warnings`.

  Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–03):**
  - Built `--offline` against the worktree, with the emulator feature, in its own build directory.
  - The board builder refuses any of the following: a non-zero exit, an unexpected refusal, a failed check, the synthetic PIN, a UUID, or a local path. It also requires every health reading named below.
- **The planted defects (04):** 8 of them, run at this head before `just pr`.
  - Each was run against the suite its rule belongs to, then restored with git.
  - The worktree was clean afterwards.
- **Builds ran without debug information** (`CARGO_PROFILE_DEV_DEBUG=0`).

**Data is synthetic throughout:** the reference chain's fixture store, with register `REG01` and a shift open on it dated 2026-11-05, cashier C-100 with one synthetic PIN, and ten bottles of oil at 1.250.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | 1.7.6b's `Done when` | the print queue's tests pass, the step's five among them | **pass**: 19 of 19, the step's five in `print_queue::unavailable` (01) |
| 2 | The IPC contract | all pass, `health_status` declared | **pass**: 7 of 7 (01) |
| 3 | Every `terminal` and `pos-hardware` test; clippy | all pass; clean | **pass**: 258 of 258, with 2 skipped (the device conformance target and CI's emupos cross-check, each `#[ignore]` by design); clippy exit 0 (01) |
| 4 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `198f25a`: 1027 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog and the frontier reconcile (01) |
| 5 | No printer: health before anyone signs in | the printer unavailable, raised as an alarm | **pass**: `"no_printer"`, 0 waiting, `["printer_unavailable"]` (02) |
| 6 | Three sales with no printer | each told at Pay, each completes, each receipt kept and queued | **pass**: `"no_printer"`, warned, three times; `"queued"` and `"no_device"` each; three jobs queued, no attempt (02) |
| 7 | Health after them, signed out | three receipts waiting, two alarms, no question | **pass**: 3 waiting, `["printer_unavailable","receipts_waiting"]` (02) |
| 8 | The register opened with a printer; one more sale | health reads it ready with three waiting; the next sale prints all four, each a stored receipt byte for byte; nothing waits after | **pass**: `"ready"`, 3 waiting, `["receipts_waiting"]`; four pages, `REG01-000001` to `-000004` in order, each check PASS; 0 waiting, no alarm (02) |
| 9 | A printer offline at the sale, back five minutes later | warned; the receipt refused before a byte and kept; health raises both; once back, the next sale prints the kept one unchanged | **pass**: `"offline"`, warned; `"refused"`, `failed:offline`; `["printer_unavailable","receipts_waiting"]`; then `"ready"`, 1 waiting; two pages, the first `REG01-000001`, check PASS; its attempts `failed:offline, printed` (03) |
| 10 | 8 planted defects | each turns the test that names its rule red | **pass**: 8 of 8 caught, each compiled (04) |

## A required check that is red, and why

**`protected-paths` is red by design.** The byte-pinned `scripts/check-test-catalog.py` moves E.85's three names, `a_sale_completes_with_no_printer_and_queues_the_artifact`, `the_missing_printer_is_an_alarm_not_a_modal` and `a_queued_artifact_prints_unchanged_once_a_printer_returns`, from `PLANNED` to `PLANNED_RETIRED`, and needs the owner's read.

## Not validated here, and why

- **The screens that show the health.** The lock screen is 1.11.4's, and the status strip is Phase 2's `StatusStrip.tsx`.
- **A kept receipt printing the moment a printer returns.** Until 1.7.14's worker drains the queue off the command path, kept receipts print with the next sale. That is what steps 8 and 9 show.
- **A print that may have come out, at the counter.** `a_print_that_may_have_come_out_waits_for_a_person` covers it on the simulator (01). The emulator's silence cannot produce one, because the driver reports `Printed` once its bytes are written (1.8.3's kit recorded the same).
- **Observed, not changed here:** with the printer offline, the finalize answers that the drawer was `opened` (03). The driver reports a drawer pulse as sent once its bytes are written, whatever the printer's state. The PR's "Read before approving" names it.
