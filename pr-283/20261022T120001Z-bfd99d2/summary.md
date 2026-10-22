**Validated at `bfd99d2`.** Microstep 1.10.1, migration `0006`, the stock ledger, is complete with this PR. It also closes #197's three guards.

**Setup**
- **Code under test:** a clean scratch worktree of this exact head.
- **Harness:** throwaway, outside the repository, built against that worktree's `pos-db` with its `fixtures` feature.
- **Database:** a fresh encrypted register database, opened through `pos_db::open`, which applies `0001`–`0006` as a register does.
- **Synthetic data only:** products, prices and costs are invented.
- **No app, printer or emulator:** this PR is the schema. The store's day is driven through the database the way the stock repository (1.10.2) and finalize (1.8.3) will.
- **The migration file itself is never edited,** not even in a scratch worktree, because the repository's immutability guard covers committed migrations. Each guard it adds is instead proved by dropping it from a scratch database of its own (steps 14–18).

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | A fresh register database opens | schema version 6, with the ledger, its cache and the adjustment intent | ✅ Board 01 |
| 2–4 | Deliver 10 pods at 1.500; sell 1; deliver 10 more at 2.100 | the sale's stock fact is in its own commit at the cost in force, 1.500; today's average moves to 1.816; the sale keeps 1.500 | ✅ Board 01 |
| 5 | Rewrite the sale's cost, then delete its event | both refused (I-6) | ✅ Board 01 |
| 6 | The cache against the ledger | on hand equals the sum of every event, and the cache counted every one | ✅ 19,000 milli over 3 events, both ways. Board 01 |
| 7 | A sale whose stock fact was never written | cannot complete | ✅ Board 01 |
| 8 | A sale of a service (an e-recharge) with no stock fact | completes: a service moves no stock | ✅ Board 02 |
| 9 | A delivery with no known cost | unflagged: refused; flagged: NULL and estimated, never a zero that reads as free | ✅ Board 02 |
| 10 | A 0.416 JOD cheese label whose derived weight is 49 g (E.80) | as a measured event, refused; flagged as derived, 49 g leaves stock and the sale completes | ✅ on hand 2,000 → 1,951 milli. This is the correction to the plan's schema the PR body explains. Board 02 |
| 11–13 | #197: a training flag of 2; a receipt artifact with no envelope; deleting a gapless counter | each refused by name, while a flag of 1 is accepted | ✅ Board 02 |
| 14–18 | Each new guard dropped from a scratch database: the stock-fact completion gate, the training flag, the receipt envelope gate, the counter's delete guard, and the ledger's update guard | refused while it stands; the same statement lands once it is gone | ✅ all five. Board 03 |
| 19 | `verify-schema` and `verify-pg-migrations` | the chain applies as the register does; the mirror conforms | ✅ The Postgres mirror was applied to a throwaway PostgreSQL container: `DATABASE_URL` was unset, and no other server was touched. Board 04 |
| 20 | The tests | `0006`'s suite, all of `pos-db`, and the terminal | ✅ 10/10, 173/173, 18/18. Board 04 |
| 21 | Three mutants of the Rust that writes alongside `0006` | each caught | ✅ No stock facts: 22 tests fail. A sale event at the wrong cost: refused by the ledger itself. A duplicate's envelope that doesn't name it: the duplicate test fails. The worktree was restored to 0 changed files. Board 05 |

CI at this head: `rust`, `printer-conformance`, `guards`, `web`, `supply-chain`, `topology`, `label`, and CodeQL with its five analyses are green, with **0 open code-scanning alerts**. **`protected-paths` is red by design.** It names one file, `scripts/check-test-catalog.py`, where one planned test name was retired.

**Not validated here, and why**
- **The stock repository and the cache rebuild command** are 1.10.2–1.10.3. Here, events are written directly, the way that repository will write them.
- **The server's stock tables.** The mirror creates nothing and records why: `stock_ledger` waits on the unsigned multi-tenant decision.
