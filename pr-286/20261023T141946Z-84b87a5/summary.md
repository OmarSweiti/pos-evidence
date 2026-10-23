**Validated at `84b87a5`.** Microstep 1.2.4's database half, `TradeScaleRepository`. 1.2.4 itself stays partially delivered: its shell half and one cart test remain.

`84b87a5` is this branch brought up to date with `development` after #287. Since the last validation the only change is #287's `ctutils` line in `Cargo.lock`, which nothing in `pos-db` depends on. Everything below was nonetheless re-run at this head, from a clean build.

**Setup**
- **Code under test:** a clean scratch worktree of this exact head.
- **Harness:** throwaway, outside the repository, built against that worktree's `pos-db` (with its `fixtures` feature) and `pos-domain`.
- **Database:** a fresh encrypted register database opened through `pos_db::open`.
- **Synthetic data only:** one invented scale (`Synthetic Scales`, `Deli-15`, `SN-0001`) and invented certificate references, each evidence hash being BLAKE3 of its reference.
- **How events were written:** every verification event went through `record_verification`, after its envelope was written through `OutboxRepository::write_commit` in the same transaction, as a register will write it.
- **How events were judged:** each stored state was read back and handed to the domain's `rule_may_activate` and `scale_pricing_allowed`.
- **No app or emulator:** this is the repository the scan handler (1.2.4's shell half, next) will call.

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | A commissioned scale with no evidence: read it, then try to switch a layout on | no current evidence; the domain and 0007 both refuse | ✅ Board 02 |
| 2 | Record the first event with no envelope, then with an envelope carrying another payload (I-9) | both refused, nothing written | ✅ Board 02 |
| 3 | Record it with its envelope | recorded; the manifest carries exactly the event's canonical payload | ✅ Boards 02 and 04 |
| 4 | Switch the layout on, read the active layouts, parse a label | accepted; one layout with its scale; item code 34567 | ✅ Board 02 |
| 5 | Record a maintenance event | current state maintenance; layout switched off by 0007; pricing refused | ✅ Boards 01 and 02 |
| 6 | Record event 4 before event 3 | refused: contiguous | ✅ Board 03 |
| 7 | Record the reverification, dated earlier than the maintenance event (I-7) | it is current: the highest owned number, not the latest date | ✅ Board 03 |
| 8 | The layouts after reverification | still off: switching one back on is a person's act | ✅ Boards 01 and 03 |
| 9 | Read the history; try to edit an event | three events in order; edit refused | ✅ Board 03 |
| 10 | Write an event straight into the table, past the repository | accepted with no envelope: the 0007 gap this PR reports | ✅ Board 03 |
| 11 | Each state through the domain, with a trusted clock, an untrusted one, and after `valid_until` | prices only while verified, effective, unexpired and trusted | ✅ Board 01 |
| 12 | The tests | the `Done when` filter, the domain rule, and all of `pos-db`, `pos-domain` and the terminal | ✅ 5/5, 3/3, 207/207, 202/202, 18/18. Board 05 |
| 13 | Nine mutants of `TradeScaleRepository` | each caught | ✅ All nine. The worktree was restored to 0 changed files. Board 06 |

CI at this head: `rust`, `printer-conformance`, `guards`, `web`, `supply-chain`, `protected-paths`, `topology`, `label`, and CodeQL with its five analyses are green, with **0 open code-scanning alerts**.

**Not validated here, and why**
- **The scan handler,** 1.2.4's shell half, is next. It needs this repository merged first.
- **Writing scale identity and label layouts.** They are reference data the server sends down; the walkthrough inserts them by hand.
- **The envelope trigger in the database.** The PR recommends adding it in the next migration; step 10 shows why.
- **A real scale or a real JSMO certificate.** None exists for this register; the evidence references are synthetic.
