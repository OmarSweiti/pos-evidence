**Validated at `5a5a21d`**, the head after development (with #279's tokio fix) was merged into this branch. Microstep 1.7.7 is **partially delivered**: this pull request is its repository half, so the evidence covers `ReceiptRepository` and not the terminal print worker.

**Setup**
- **Code under test:** a clean scratch worktree of this exact head.
- **Harness:** throwaway, outside the repository. It is built against that worktree's `pos-db` and `pos-domain`, and reuses the test suite's own fixture (`tests/common/registered_chain.rs`) so the sale is completed through `0005`'s gates.
- **Database:** a fresh encrypted SQLite file in a temporary directory, holding synthetic data only.
- **Time and ids:** instants are fixed synthetic arguments, and ids are fixed UUIDs.
- **No printer or emulator is involved.** This half writes and reads the queue; driving a printer from it is the worker's job, in the other half.

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | A sale completes with its original receipt and first job, written by the repository | completed; one original, hash = BLAKE3(bytes); one job, queued, 0 attempts | ✅ as expected. Board 01 |
| 2–5 | Lease, a refusal before any byte (`paper_out`), then leases at t+29 and t+30 | leased; failed with retry at t+30; nothing at t+29; leased again at t+30 | ✅ all four. Board 01 |
| 6–7 | The device goes silent mid-document, then a lease a year later | `unknown`, listed as awaiting an answer; never offered again | ✅ both. Board 01 |
| 8 | A worker appends a result to the unclaimed `unknown` job | refused by the schema | ✅ `print attempts must append once and match the claimed job transition`. Board 02 |
| 9 | A person answers that a readable receipt came out | `cancelled`; nothing awaiting, nothing unprinted | ✅ Board 02 |
| 10–12 | A worker dies holding a second sale's job; its lease lapses; someone tries to requeue it; it is resolved | listed by `expired_leases`; requeue refused; `unknown` (`lease_expired`) | ✅ all three. Board 02 |
| 13 | The first sale after every outcome (I-4) | every column of its row unchanged | ✅ 25 of 25 columns identical. Board 02 |
| 14 | Close and reopen the database | the bytes, both jobs and every attempt unchanged | ✅ Board 02 |
| 15 | `cargo nextest run -p pos-db --test receipt`, then `-p pos-db` | 10/10; the whole crate green | ✅ 10/10 and 162/162. Board 03 |
| 16 | Seven mutants of `receipt.rs`, each applied, tested and restored | every mutant fails at least one test | ✅ 7 of 7 caught. The worktree was restored to 0 changed files. Board 04 |

CI at this head: `rust`, `printer-conformance`, `guards`, `web`, `supply-chain`, `protected-paths`, `topology`, `label` and CodeQL are all green. `supply-chain` was red on the previous head, `e560280`, only because of the yanked tokio 1.54.0. #279 fixed that, and this merge brings the fix in.

**Not validated here, and why**
- **The terminal's print worker, `print_resolve_unknown` and the duplicate path's payload.** They are 1.7.7's other half and are not written yet. #280 records the two decisions they need.
- **A real or emulated printer.** Nothing in this half talks to one.
