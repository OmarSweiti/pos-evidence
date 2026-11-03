# Validation: 1.10.4, negative stock allowed and flagged, or blocked before tender

**Commit validated:** `4f02261`, the pull request's head, checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **`pos_domain::NegativeStockPolicy::screen`** sums a basket's lines per product and returns every product the basket would leave below zero. That list is the flag. Under `HardBlock`, a non-empty flag is refused as `StockError::WouldGoNegative`.
- **`StockRepository::negative_stock_policy`** reads `store.allow_negative_stock`. A value other than 0 or 1 is refused.
- **`StockRepository::screen_basket`** screens a basket against the cache before tender. It skips services and writes nothing.

No screen or command calls these yet; the cart commands are 1.4.11's. The runtime is therefore shown through the repository itself, on synthetic register files.

**How it ran:**
- **The suites (01).** The `Done when` command, `cargo nextest run -p pos-db repo::stock::tests::`; the domain's stock tests; the new property again at `PROPTEST_CASES=20000`; every `pos-db` test; clippy; the whole workspace; and the test catalog.
- **The harness (02, 03, 04).**
  - It is linked against this head's `pos-db` and `pos-domain` and built `--offline` from the worktree's `Cargo.lock`.
  - Every step is its own process on one register file, with its exit status recorded.
  - A sale is made the way 1.8.3's finalize will make it: a domain cart, priced under the store's pack and policy and paid by card. Then the sale, its stock events (none for a service), its receipt and its audit row go under one envelope in one transaction, through `SaleRepository` and `StockRepository`.
  - `sell` never screens, as finalize never will.
- **The planted defects (05).** 12 defects, each applied to a scratch worktree at `4f02261`, tested, and restored. No migration was mutated.

Data is synthetic throughout: an invented store, two registers (`REG01`, `REG02`) and three products (`SYN-OIL`, `SYN-BREAD`, and `SYN-DELIVERY`, a service).

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db repo::stock::tests::` | 11 pass | **pass**: 11 of 11, the plan's three among them (01) |
| 2 | The domain's stock tests; the property at 20,000 cases; `pos-db`; clippy; the workspace; the test catalog | all pass | **pass**: 17 of 17; 1 of 1 at 20,000 cases; 297 of 297; clippy clean; 839 passed, 4 skipped (the existing `#[ignore]` ones); the catalog reconciles with E.12's Phase 1 name retired (01) |
| 3 | The default store, two oils on hand: a basket of 1 + 2 oils, a bread and a delivery is screened | the oil is flagged, nothing refused, nothing written | **pass**: `FLAG SYN-OIL: on hand 2.000, the basket takes 3.000`; `AllowAndFlag`; 0 rows written (02) |
| 4 | The same basket priced, paid and finalized | the sale completes, its oil event records the position below zero, and the report lists the oil | **pass**: `REG01-000001` completed at 5.100 JOD. Its second oil event leaves -1.000, and the report lists `SYN-OIL -1.000`. The delivery has no stock event (02) |
| 5 | A hard-block store, two oils on hand: the two oils screened, then 2 + 1 oils, then a service with the last breads | the last units pass; the second oil line is refused before tender, writing nothing; the service is never screened | **pass**: no flag; `refused WouldGoNegative` naming on-hand 2.000 and 3.000 taken, 0 rows written; no flag (03) |
| 6 | Under the hard block, two sales already paid for: two oils, then one more | both complete; the second leaves -1.000, flagged | **pass**: `REG01-000001` leaves 0.000, and `REG01-000002` leaves -1.000, flagged and listed by the report (03) |
| 7 | `store.allow_negative_stock` written by hand as 2, then a screen | refused by name, not read as a guess | **pass**: `StoredStockRecordInvalid` from both the policy read and the screen, with 0 rows written (03) |
| 8 | E.12: REG01's and REG02's files, each with the hard block and the last oil in its own ledger, each screening and selling it | both screens pass and both sales complete | **pass**: `REG01-000001` and `REG02-000001` both completed, each leaving its own file at 0.000 (04) |
| 9 | 12 planted defects | each turns the test that names its rule red | **pass**: 12 of 12 caught. Every mutant compiled, and the worktree was restored with no changed file (05) |

`just pre-push` was run by `just pr` before the pull request was opened, and passed: lint, tests, the web build, guards and the secret scan.

## Not validated, and why

- **A register calling the screen.** The cart commands that will call `screen_basket` are 1.4.11's, and the note there says how. `cart_set_qty` and `tender_begin` have no owning microstep yet, and the pull request says so.
- **The store's stock going negative where two registers' ledgers meet.** That is Phase 3's half of E.12, `both_sales_of_the_last_unit_stand_and_stock_goes_negative_flagged`. Here each register's own file is shown.
- **The required `protected-paths` check is red by design.** This pull request retires its `PLANNED` test name in the byte-pinned `scripts/check-test-catalog.py`, as every earlier retirement did.
