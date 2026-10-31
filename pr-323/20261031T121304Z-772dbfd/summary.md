# Validation: microstep 1.4.9, `price_cart`

**Commit validated:** `772dbfd`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- **A pure change:** `price_cart` is a pure function. No database, device, emulator or screen is involved.
- **A throwaway harness**, not in the repository, rang carts up through the pull request's own transitions and priced them with its own `price_cart`, under a synthetic tax pack: 16% standard, zero-rated and exempt. Its sweep compares every result with a pricing written separately in the harness. That pricing takes each line's discounts from 1.4.6's `discount_attributions`, which #319's evidence checked, and states everything else on its own: the gross, the tax under each price mode and rounding rule, the totals and the receipt rows. It ran as a release build. Every id, name, reason code and amount is synthetic.
- **Planted defects:** each one changed `price_cart` in `crates/pos-domain/src/cart.rs` in the worktree, ran the cart and pricing suites, and was restored, proptest seed files included. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain cart::tests::prop_ && cargo test -p pos-domain --bench price_cart` (this step's `Done when`) | every property passes, and the 200-line fixture is priced once | **pass** (01) |
| 2 | This step's twelve tests, the domain crate, the workspace | no failure | **pass**: 330/330; 729/729 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | The four properties alone, release build, `PROPTEST_CASES=65536` | pass | **pass** (01) |
| 4 | One timed run of the benchmark | a figure, indicative only | about 87 µs per 200-line recompute on a development laptop, criterion's estimate; not a baseline (01) |
| 5 | Two teas at 1.250 under 16%, prices including and excluding tax | 2.500 holding 0.345 tax; 2.500 plus 0.400 | **pass** (02) |
| 6 | The teas and three packs, 5% off the teas, then 10% off the basket | discounts 0.125, 0.238 and 0.375 taken off before tax; 6.250 − 0.738 + 0.882 = 6.394 excluding tax, 5.512 including it | **pass** (02) |
| 7 | A standard, a zero-rated and an exempt line | one receipt row for each, tax on the standard one only | **pass** (02) |
| 8 | The teas overridden to 1.000 | priced at 2.000, no discount line | **pass** (02) |
| 9 | A missing band, an unregistered store, an export supply, one line id twice, a stored line discounted past its value | refused by name, except the unregistered store, which prices with no tax and no band | **pass** (02) |
| 10 | 100% off the basket (E.18) | a total, tax and every line of zero | **pass** (02) |
| 11 | A weighed cheese, the teas and a zero-rated pack, 0.010 off the basket, in all six scan orders (E.19b) | the same total, tax and discounts every time | **pass** (02) |
| 12 | Every cart of two lines from six prices, five quantities, three bands and four line discounts, with four basket discounts, under four rules, both price modes, registered and not; and each re-priced with its lines in the other order and fresh ids | 0 differing from the pricing stated separately, and 0 re-pricings differing | **pass**: 4,158,720 pricings and 4,158,720 re-pricings, 0 wrong (03) |
| 13 | Nineteen planted defects | each caught | **pass** (04): all nineteen caught, worktree restored |

## Notes

- **All nineteen planted defects compiled**, and their anchors were checked unique against this head before the run. Writing them showed that no test reached the refusal of a line worth less than nothing, because the transitions never build one. `a_line_worth_less_than_nothing_is_refused` was added before the pull request was opened, and it catches that defect.
- **One defect makes a line's position change its rounding.** It was caught by the reordering property, which is what E.19b asks that property to do.
- **The sweep counts 287,253 discounts the transitions refused**, such as 0.001 off a line worth nothing. Those carts were priced without the refused discount, as a till would leave it out.

## Not validated, and why

- **The benchmark's budget**, a p99 under 16 ms on the reference register: no reference register exists before milestone 5.8, and 5.8.2 proves it.
- **The payment-stage transitions** (1.4.8), which take a `PricedCart`, and **how the fiscal document shows an override** (2.7.3): neither exists yet.
