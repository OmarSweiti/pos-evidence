# Validation: microstep 1.4.7, the price override

**Commit validated:** `c6eec24`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- **A pure change:** `override_price`, `add_line`'s ceiling and `StockPosition::cost_basis` are pure functions. No database, device, emulator or screen is involved.
- **A throwaway harness**, not in the repository, called the pull request's own functions under the shipped role grid: a shift lead and a manager hold `price.override`, a cashier does not. Its sweep compares every outcome with the owner's rules written separately in the harness. It ran as a release build. Every id, name, reason code and amount is synthetic.
- **Planted defects:** each one changed `crates/pos-domain/src/cart.rs`, `pricing.rs` or `stock.rs` in the worktree, ran the cart, pricing and stock suites, and was restored, proptest seed files included. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain cart::tests:: pricing::tests:: stock::tests::` (this step's `Done when`) | this step's fourteen tests pass, and 72/72 | **pass** (01) |
| 2 | The domain crate, then the workspace | no failure | **pass**: 318/318; 717/717 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | The property alone, release build, `PROPTEST_CASES=65536` | pass | **pass** (01) |
| 4 | Two teas at 1.250 costing 1.000: the shift lead at 0.999 and 1.000, the manager at 0.999, an escalated approval at 0.999; the cashier asks for proof | 0.999 refused for all three as below the floor; 1.000 accepted and audited with its floor; the cashier holds no `price.override` | **pass** (02) |
| 5 | A product delivered with no cost, read through the ledger: lowered, raised, put back | no cost basis; 1.249 refused as no recorded cost; 1.300 and 1.250 accepted, the second clearing the override | **pass** (02) |
| 6 | The catalogue sells below its cost of 1.400: 1.300, then 1.249 | 1.300 accepted with no floor applied; 1.249 refused at the floor | **pass** (02) |
| 7 | The shelf says 0.990, the till 1.250, the cost 1.000: as `displayed_price`, as the typed code, with no recorded cost; then 1.250 and 1.300 | all three accepted below the floor, each with a label to print again; 1.250 and 1.300 refused as not lower | **pass** (02) |
| 8 | A staple at its ceiling of 1.250: entering; 1.251 by the shift lead, the manager and with an approval; a product priced above it; its price labels at 1.300 and 1.200; a weighed product's label; that label's line overridden | enters at its ceiling; every 1.251 refused; the over-priced product and the 1.300 label refused; the 1.200 label enters with its ceiling; the weighed label enters with none; its line refuses an override | **pass** (02) |
| 9 | The teas overridden to 1.000, then the cashier's 0.100 and 0.101 off; an amount off the basket through a raise and a lower price | 0.100 accepted on a line worth 2.000, 0.101 above the 5% cap; the basket's amount kept on the raise, cleared on the lower price | **pass** (02) |
| 10 | Every line priced 0.000–0.012 under every ceiling; every override from every price it could charge, to every price −0.001–0.015, for a reason or as the shelf's price, under every cost; every price label on a counted and a weighed product | 0 differing from the rules stated separately | **pass**: 208 entries, 1,654,848 overrides and 448 labels, 0 wrong (03) |
| 11 | Twenty-eight planted defects | each caught | **pass** (04): all twenty-eight caught, worktree restored |

## Notes

- **The planted defects all compiled** and each was caught by a test aimed at it, listed on board 04. Their anchors were checked unique against this head before the run.
- **Board 01's first block keeps this step's fourteen tests.** The listing's name filter also matched three earlier tests, one cart and two stock, which passed in the same run and are left out of that block.
- **Before the pull request, one local run of the whole domain crate failed `an_add_line_never_arrives_from_outside_with_its_price` once**, the compile-fail fixture for `AddLine`. It passed in every later run, including the gate's, CI's and this one, and a deliberately cold rerun did not reproduce it. The likeliest cause, not confirmed: that run had set `CARGO_TARGET_DIR` to a relative path, so the fixture harness built into a fresh directory inside the crate, and the two tests in that file compiled into that cold, empty directory at once. The failing run's detail was not kept.

## Not validated, and why

- **The `cart_override_price` command** (1.4.11), which reads the cost basis and binds an escalated approval to the line and the price: it does not exist yet.
- **How the priced cart shows an override** (1.4.9), and **the label worklist** that stores the reprint (4.6.2, migration `0013`): neither exists yet.
