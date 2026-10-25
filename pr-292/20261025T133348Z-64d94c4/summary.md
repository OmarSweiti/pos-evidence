# Validation — microstep 1.10.2, the stock repository

**Commit validated:** `64d94c4`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- A throwaway harness, not in the repository, opened a **fresh encrypted register database** with `pos_db::open`, seeded the shared registered-chain fixture, and drove the pull request's own `StockRepository`. Each step was one transaction: prepare, envelope, the audit row where the step is audited, then append.
- The products are synthetic.
- There is no stock screen or command yet: 1.10.5 adds them, so no emulator or app window is involved.
- Each mutant edited one line in the worktree and was then restored; the run ends with "restored: 0 changed files".

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db repo::stock::tests::` (the Done when) | all pass, including the fault injected before each write | **pass**: 8/8 (01) |
| 2 | The domain rule and both crates, then the workspace | no failure | **pass**: 10/10; 432/432; 592/592 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | A day of stock on a fresh database | every event records the cost basis it captured and the stock level it leaves | **pass** (02); details below |
| 4 | What travels with the event | the envelope carries exactly the event; a missing or different envelope is refused, with nothing written | **pass** (03): the payload is byte-equal to `PreparedStockEvent::payload()`; both refusals left every table's row count unchanged |
| 5 | A fault before each of the seven writes | every companion row rolls back | **pass** (04): all seven left every table unchanged; without a fault, the same transaction commits whole |
| 6 | The cache, zeroed, then rebuilt | identical to the trigger's projection | **pass** (05): 3 rows written, identical |
| 7 | Six one-line defects | each caught by a test | **pass** (06): all six caught; worktree restored |

### Step 3 in detail (board 02)

- **Opening stock:** 24 colas at an observed 0.380 JOD set the WAC to 0.380. Rice with no known cost stored its cost as none, not zero, and marked it estimated.
- **The sale:** two cola lines and one rice line became three consecutive events. Each recorded the cost in force (0.380, or 0.000 estimated for the rice).
- **Waste and a count:** waste left the WAC unchanged. A count correction at a projected 0.380 kept the WAC and marked it estimated.
- **A delivery at 0.500 into positive stock** was refused by name, and nothing was written. This is the designed refusal until 4.2.2.
- **Rice written off past what was on hand** went to −2, and `negative_stock` lists it.

## Found and fixed during this validation

The first evidence run, at `760cc28`, showed the blend refusal's message printing internal type dumps (`Money { minor: 500, currency: Currency { code: [74, 79, 68] … } }`). Commit `64d94c4` makes it read *"blending a unit cost of 500 into positive stock valued at 380 (JOD, in minor units) …"*, adds a test that pins the text, and all of this evidence was re-run at that head.

## Not validated, and why

- **Completing a sale.** The sale here stays parked. Completing one belongs to 1.8.3; `0006`'s completion gates were proven at 1.10.1.
- **Average-cost blending.** Its arrival belongs to 4.2.2; here it is refused, as shown in step 3.
- **The negative-stock flag at checkout and the per-store hard block.** These belong to 1.10.4.
- **Adjustments' reason codes and notes, and any screen.** These belong to 1.10.5.
- **Windows and Linux.** Only macOS (aarch64) ran here; CI ran the Linux suites.
