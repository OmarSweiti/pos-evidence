# Validation: 1.10.5's domain and database half, approved stock adjustments and opening stock

**Commit validated:** `2fb86d2`, the pull request's head, checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **`StockAdjustmentRepository`** turns a stock adjustment into an event. `prepare` stores the request under the event id it will carry, with its digest. `binding` and `prepare_effect` hold an approval to that exact request. `commit` writes the event, its audit row and the approval's consumption, then removes the request.
- **`pos_db::seed::opening_stock`** reads a three-column opening-stock file and posts each row as a ledger event.
- **`StockRepository::position`** answers what `stock_on_hand` will.
- **In the domain:** the five adjustment reasons and their directions, `StockSource::Adjustment`, and the canonical prepared-intent digest.

No screen or command calls these yet. `stock_on_hand`, `stock_adjust_prepare`, `stock_adjust` and `StockAdjust.tsx` are this step's deferred half, under its `Full-step status:` marker. The runtime is therefore shown through the repository itself, on synthetic register files.

**How it ran:**
- **The suites (01).**
  - The `Done when`'s `pos-db` half: `cargo nextest run -p pos-db opening_stock_` and `--test stock_adjustment`.
  - The domain's adjustment and digest tests.
  - The properties again at `PROPTEST_CASES=20000`, with 1.10.3's cache property, whose `adjust` events now carry a reason.
  - Every `pos-db` test, clippy, the whole workspace, the test catalog, and the frontier.
- **The harness (02, 03, 04).**
  - It is linked against this head's `pos-db` and `pos-domain` and built `--offline` from the worktree's `Cargo.lock`.
  - Every step is its own process on one register file. A step that dies calls `abort()`.
  - A sale is made the way 1.8.3's finalize will make it.
  - The manager (`MGR-1`) asks and the owner (`OWN-1`) approves, each with their role's own grants.
  - No digest is printed, because `content_hash` is a `_hash` field under the never-log rule.
- **The planted defects (05).** There were 18 defects, each applied to a scratch worktree at `c48f516`, tested and restored. The only commit after `c48f516` is `2fb86d2`, which changes how three approval nonces in `tests/stock_adjustment.rs` are derived: from the bound id rather than a literal, which CodeQL's hard-coded-value rule requires. It changes no code under test. No migration was mutated.

Data is synthetic throughout: an invented store, register `REG01`, users `MGR-1` and `OWN-1`, and products `SYN-OIL`, `SYN-BREAD` and `SYN-RICE`.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | The `Done when`'s `pos-db` half, and the suite | all pass | **pass**: `opening_stock_` 3 of 3; `--test stock_adjustment` 9 of 9 (01) |
| 2 | The domain's tests; the properties at 20,000 cases; `pos-db`; clippy; the workspace; the catalog and the frontier | all pass | **pass**: 7 of 7; 3 of 3 at 20,000 cases; 305 of 305; clippy clean; 854 passed, 4 skipped (the existing `#[ignore]` ones); both checks reconcile (01) |
| 3 | An opening-stock file of three rows, one with no cost, read and loaded | three `opening_stock` events under one envelope; the cache projected from them; the costless row stored `NULL`, estimated | **pass**: oil 24.000 at 0.800, bread 40.000 at 0.220, rice 10.000 at `NULL`, estimated (02) |
| 4 | A sale of one oil | it completes and captures the cost in force | **pass**: `REG01-000001`, 1.250 JOD; its event's cost is 0.800 (02, 03) |
| 5 | The manager asks for 2 oils written off as damage, the owner approves, and the effect dies inside its transaction | the next process finds the request pending, the approval unspent and no event | **pass**: exit 134 after writing everything; then `0x30 damage -2.000` pending, `0x50 for 0x30 (unspent)`, no audit row, oil still 23.000 (03) |
| 6 | The effect again | it commits: the event, the audit row and the consumption share the request's id, and the request is gone | **pass**: oil 21.000. The `0x30` event has reason `damage` and note "crate dropped". The audit row records actor `MGR-1`, approver `OWN-1`, approval `0x50` and `amount_minor` 0, and the approval shows consumed. The sale's cost is still 0.800 (03) |
| 7 | Rice found by a count, approved and posted | it arrives at the cost basis in force, which for rice, never costed, is unknown | **pass**: rice 13.000; its event's cost is `NULL`, estimated (03) |
| 8 | An unknown reason code; damage that would add stock | refused by name; nothing stored | **pass**: `UnknownAdjustmentReason("correction")`; `AdjustmentAgainstItsReason { reason: Damage }` (04) |
| 9 | An approval for `0x42` spent on `0x43` | refused by the repository | **pass**: `ApprovalRefused(ApprovalEntityMismatch(0x42, 0x43))` (04) |
| 10 | A request changed after approval: by `UPDATE`; then with the freeze trigger dropped by hand; then the file reopened | refused by the trigger; then by the recomputed digest; then the file refused at open | **pass**: "prepared stock intent is immutable after approval"; then `StockAdjustmentAltered(0x60)`; then `SchemaShapeMismatch` naming the missing trigger (04) |
| 11 | 18 planted defects | each named defect turns its test red; M15 predicted to survive | **pass**: 17 of 17 caught, and M15 survived as predicted. Every mutant compiled, and the worktree was restored with no changed file (05) |

`just pre-push` was run by `just pr` before the pull request was opened, and passed: lint, tests, the web build, guards and the secret scan. CodeQL then flagged the six literal nonces that `2fb86d2` replaces.

## Not validated, and why

- **The commands and the screen.** `cargo nextest run -p terminal commands::stock::tests::` and `vitest run src/screens/StockAdjust.test.tsx` are the `Done when`'s other half. They need the register's database, the signed-in user and stored-approval resolution in the shell's state, which no microstep's `Files:` line yet creates. The step's `Full-step status:` marker says how each command calls this half.
- **The minimarket's real opening-stock file.** It names 1.12.1's seed products and lands with them. The harness reads a synthetic file through the same reader.
