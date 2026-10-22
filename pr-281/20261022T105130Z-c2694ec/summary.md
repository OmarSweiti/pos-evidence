**Validated at `c2694ec`.** Microstep 1.7.7 is complete with this PR: the register's print worker and a person's answer for an ambiguous print.

**Setup**
- **Code under test:** a clean scratch worktree of this exact head.
- **Harness:** throwaway, outside the repository, built against that worktree's `terminal`, `pos-db` (with its `fixtures` feature) and `pos-hardware` (with `emulator`).
- **Database:** a fresh encrypted SQLite file holding synthetic data only. Instants are fixed synthetic times, and ids come from a deterministic source.
- **Printer:** 1.7.12's emulated network printer, loaded with the seeded `epson-tm-t20iii-ethernet-80mm` profile and listening on 127.0.0.1. The worker reaches it through the register's own network driver (1.7.10), as a register would.
- **One exception:** step 5's ambiguous outcome comes from the in-process simulator, bound to the same profile. On loopback, a device that goes silent takes a receipt this size into socket buffers, and the driver honestly reports it printed. TCP cannot produce this outcome for a document this small.
- **Audit rows:** each sale's completion row is written through `AuditRepository`, so the register's audit chain is real end to end. The shared fixture's own audit rows carry invented hashes.

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | A sale completes; `queue_original` writes its original and one job | completed; one job, queued, 0 attempts | ✅ Board 02 |
| 2 | The worker drains onto the emulated TM-T20III | printed; the page is the receipt dot for dot; no drawer pulse | ✅ Board 02 |
| 3–4 | Paper out, then passes at t+39 s and t+40 s | failed with a retry at t+40 s; nothing at t+39 s; printed at t+40 s | ✅ Board 02 |
| 5–6 | A device goes silent mid-document, then 24 hourly passes on a healthy printer | `unknown`, awaiting a person; never sent again; no drawer pulse | ✅ Board 02 |
| 7 | A cashier answers "no readable receipt" with another sale's document | refused, nothing written | ✅ `ModelDiffersFromOriginal`; 3 → 3 artifacts. Board 02 |
| 8 | The same cashier answers with this sale's own document | a duplicate citing the original, with its own job; the original job cancelled; one envelope `[receipt_artifact, audit_log]` | ✅ Board 03 |
| 9 | The worker prints the duplicate | the page is the receipt with the DUPLICATE watermark, dot for dot | ✅ Board 01 shows both pages; Board 03 |
| 10–11 | A worker dies holding a job; the database is closed and reopened; the worker drains past the lease | everything reads back unchanged; the lapsed lease becomes `unknown` and is never re-sent | ✅ Board 03 |
| 12 | Every completed sale after every outcome (I-4) | every column unchanged | ✅ 4 sales × 25 columns. Board 03 |
| 13 | The register's audit chain | the answer is a chained row and the chain verifies | ✅ `IntactUnanchoredFrom { entries: 5, unanchored_from: 0 }`. Every row verifies; it is unanchored only because no Z close has written an anchor. Board 03 |
| 14 | `cargo nextest run -p terminal print_queue::`, `-p pos-db`, and both shipped-graph guards | all green | ✅ 14/14, 163/163, 1/1, 1/1. Board 04 |
| 15 | Twelve mutants, each breaking one rule and then restored | every mutant fails at least one test | ✅ 12 of 12 caught (15 failures). The worktree was restored to 0 changed files. The "not unknown" mutant removes both state checks, the one before the transaction and the one inside it. Boards 05–06 |

CI at this head: `rust`, `printer-conformance`, `guards`, `web`, `supply-chain`, `topology`, `label`, and CodeQL with its five analyses are green, with **0 open code-scanning alerts**. The previous head drew one, `rust/hard-coded-cryptographic-value` on a test's literal approval nonce; `c2694ec` derives it from the job's id instead. **`protected-paths` is red by design.** It names one file, `scripts/check-test-catalog.py`, where two planned test names were retired. The PR body explains it.

**Not validated here, and why**
- **The Tauri command and starting the worker when the register opens.** Neither exists yet; the PR body says what each waits on. The worker and the resolver were driven directly.
- **A partial page through the network driver.** The emulator has no fault that closes a connection mid-document. The unit tests cover `partial` through the simulator, and so does the first mutant.
- **A real printer.** No device is bought before the hardware gate (§6b).
- **An intermittent marker, not explained:** three times while this was developed, nextest marked a passing terminal test "leaky", meaning output was held open briefly after the test exited. It did not recur in eight consecutive runs afterwards, nor in this head's suite and its eleven mutant runs. Earlier `pos-db` mutant runs showed the same marker on tests that spawn nothing, which points at this machine rather than this code, but that is a hypothesis, not a finding.
