# Validation: microstep 1.4.12, the domain half — goods with no product, sold through a department

**Commit validated:** `d7513c5`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- **A pure change:** the transition is a pure function. No database, device, emulator or screen is involved.
- **A throwaway harness**, not in the repository, rang carts up through the pull request's own `add_line`, sold goods through a department with its own `add_department_line`, and priced, paid for and completed the result with its own transitions. It used a synthetic pack in which the department's band is zero-rated and the teas' is 16%. Its sweep compares every typed amount with the department rules, stated separately in the harness. It ran as a release build. Every id, name, code and amount is synthetic.
- **Planted defects:** each one changed one place in `crates/pos-domain/src/cart.rs` or `crates/pos-domain/src/catalog/scan.rs` in the worktree, ran the cart and pricing suites, and was restored, proptest seed files included. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain department_` (the `Done when`'s first half, the domain's) | the seven domain tests pass | **pass**: 7/7 (01) |
| 2 | The cart suite, the domain crate, the workspace | no failure | **pass**: 87/87; 356/356; 755/755 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | Every cart property alone, release build, `PROPTEST_CASES=65536` | pass | **pass**: 11 of 11 (01) |
| 4 | An unknown code beside two teas, then 2.500 sold through a department | the scan is no product, `add_line` refuses the typed amount, `add_department_line` takes it as one line with no product, in the department's zero-rated band, audited as `sale.department` with the scanned code | **pass** (02) |
| 5 | That line priced, then paid by card and completed | its own receipt row at 0%, beside the teas' 16%; completed with the department's name and no product | **pass** (02) |
| 6 | The line made three, overridden, and discounted 10% | the first two refused; the discount taken and audited as `discount.manual` | **pass** (02) |
| 7 | Above the 5.000 threshold: no approval, the right one, one for another amount or sale; the threshold itself; above and at the 10.000 cap | refused without approval and for any other binding; at the threshold, none needed; above the cap, refused even approved | **pass** (02) |
| 8 | A department not in use, nothing, less than nothing, dollars, a catalogue line, a department line made three | each refused by name | **pass** (02) |
| 9 | The shipped grid, asked for `sale.department` | cashier, shift lead and manager granted; the owner denied | **pass** (02) |
| 10 | Every amount from −0.002 to 12.000 to the fil, under three thresholds, with no approval, the right one, and one a fil off | 0 differing from the rules stated separately | **pass**: 108,027 cases, 0 differing (02) |
| 11 | Nineteen planted defects | each caught | **pass** (03): all nineteen caught, worktree restored |

## Notes

- **The planted defects found a gap, closed before the pull request opened.** `add_department_line` checks eight things about the line it is given. Listing the defects showed that only three of them had a test. `a_department_sale_is_refused_by_name` now changes each of the eight on its own, and catches each defect.
- **A discount on a department line is allowed**, by whoever holds `discount.manual`, within their cap, and audited. Nothing in §6.5 refuses it, and it is not a typed amount.
- **Nothing stores a department line yet.** `category` has no tax category, `sale_line.product_id` is `NOT NULL`, and no table holds the policy. #330 asks how that changes, and the pull request names this under "Read before approving".

## Not validated, and why

- **The terminal half:** `cart_add_department_sale` and `queue_never_stalls_on_unknown_code`, which live in 1.4.11's command file. 1.4.12 carries a `Full-step status:` marker until they land.
- **Storing a department, its line or its policy** (#330), and **the finalize transaction's missing stock event** (1.8.3).
