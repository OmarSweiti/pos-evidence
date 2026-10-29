# Validation: microstep 1.4.6, the basket discount and its division among the lines

**Commit validated:** `f1cb27c`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- **A pure change:** `apply_basket_discount`, `discount_attributions` and `prorate` are pure functions. No database, device, emulator or screen is involved.
- **A throwaway harness**, not in the repository, called the pull request's own functions under the caps the owner adopted: a cashier at 5%, a manager with none. Its sweep compares `prorate` with a largest-remainder computation written separately in the harness. It ran as a release build. Every id, name, reason code and amount is synthetic.
- **Planted defects:** each one changed `crates/pos-domain/src/pricing.rs` or `cart.rs` in the worktree, ran the cart and pricing suites, and was restored, proptest seed files included. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain pricing::tests::prop_` (the plan's `Done when`), then `cart::tests:: pricing::tests::` | 3/3, then 48/48 | **pass** (01) |
| 2 | The domain crate, then the workspace | no failure | **pass**: 304/304; 703/703 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | The three properties alone, release build, `PROPTEST_CASES=65536` | pass | **pass** (01) |
| 4 | The cashier on a basket worth 6.250: 5%, 0.312, 0.313, 20% | 5% (313 fils) and 0.312 accepted, audited on the sale; 0.313 and 20% refused as above the cap | **pass** (02) |
| 5 | 20% with a manager's approval for the sale and 1.250, then with one for a line | accepted, naming the manager and the handle; refused, naming the line | **pass** (02) |
| 6 | 0.125 off the teas, then 5% off the basket; then the manager at 6.125 and 6.126 | both accepted; attributions 0.125 on the teas, then 0.119 and 0.187 (306 fils, the leftover to the larger remainder); 6.125 accepted, 6.126 refused | **pass** (02) |
| 7 | Three lines at three tax rates, worth 0.500 each, in all six scan orders, 0.100 off | the lowest tax category takes the leftover fil in every order | **pass** (02) |
| 8 | 0.500 off the basket, then a void, a lower quantity, a line discount, an age decline; a higher quantity, another line; 10% off and a void | the first four clear it; the next two keep it; the share is kept | **pass** (02) |
| 9 | Every basket of three lines from eight values and three kinds of line, at every discount it can carry, against an independent computation; every seventh in all six scan orders with fresh ids | 0 wrong, 0 charged differently | **pass**: 1,885,248 divisions and 1,651,590 re-divisions (03) |
| 10 | Twenty planted defects | each caught | **pass** (04): all twenty caught, worktree restored |

## Notes

- **The harness found a defect, now fixed at this head.** At `1b893ee`, the scan-order sweep charged 1,290 of 1,651,590 re-divisions differently. Two lines of one product at one price and quantity can be worth different amounts, their own discounts differing, and still be cut equally. The tie-break key could not tell them apart, so the leftover fil fell to scan position. `f1cb27c` ranks by the key and then by the line's value, and the sweep now finds 0. The properties had generated a value with each kind of line, so they never produced the case. They now draw kind and value separately, mostly a few fils, and fail against the old rule.
- **Earlier runs at `48b0bb1` and `1b893ee` found test gaps,** each since closed. One planted defect, dropping quantity from the key, survived until a test of two lines alike but for quantity was added. One planted defect did not compile until it was rewritten.
- **nextest reported one passing test "leaky" twice:** once in the step's command at `1b893ee`, and once in one planted defect's run here. The term means a passing test's process held its output open briefly after it ended. Three repeat runs of the step's command, and the step's command at this head, showed none. The pure domain starts no process.

## Not validated, and why

- **Pricing the cart** (1.4.9), which reads `discount_attributions`, and **the `cart_discount_basket` command** (1.4.11): neither exists yet.
