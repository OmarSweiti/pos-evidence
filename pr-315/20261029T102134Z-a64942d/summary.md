# Validation: microstep 1.4.5, its line-discount half

**Commit validated:** `a64942d`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- `apply_line_discount` and `pricing.rs` are pure: no database, device, emulator or screen is involved. Approvals were issued with the domain's own `authorize` and `ApprovalHandle::issue`, under the shipped role grid. The caps are the ones the owner adopted: 5% for the cashier, none for the manager.
- A throwaway harness, not in the repository, called the pull request's own functions and printed what they returned. It ran as a release build. Every id, name, reason code and amount is synthetic.
- Each planted defect changed `crates/pos-domain/src/cart.rs` or `pricing.rs` in the worktree, ran the cart and pricing suites, and was restored, proptest seed files included. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain cart::tests:: pricing::tests::` (the `Done when` this PR gives the line half) | 36/36 | **pass** (01) |
| 2 | The domain crate, then the workspace | no failure | **pass**: 292/292; 687/687 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | `prop_discount_never_makes_a_line_negative` alone, release build, `PROPTEST_CASES=65536` | pass | **pass** (01) |
| 4 | The cashier (5% cap) on a line worth 2.500: 5%, 0.125, 5.0001%, 0.126 | the first two accepted with no approver; the last two refused as above the cap | **pass** (02) |
| 5 | 20% (0.500) without approval, then with a manager's handle for this cashier, line, 500 fils and reason | refused, then accepted; the line and the intent name the manager and the handle, and the intent's `amount_minor` is 500 | **pass** (02) |
| 6 | That handle expired, and handles for 400 fils, another line, another reason, another cashier | each refused by name | **pass** (02) |
| 7 | A handle offered for a discount within the cap | accepted, and the handle is not named, so it is not spent | **pass** (02) |
| 8 | 5% then 5% again, then zero; 0.125 off then one tea instead of two; 5% then five teas | one 5% discount, then none; the quantity change refused; the share kept | **pass** (02) |
| 9 | The manager (no cap): 100%, 100.0001%, 2.500, 2.501 off 2.500; 0.665 and 0.666 off cheese worth exactly 0.665667; −1%; 10.00 USD | 100%, 2.500 and 0.665 accepted; 0.666 refused, although the line rounds to 0.666; the rest refused by name | **pass** (02) |
| 10 | 173,298 requests: every share from 0% to 101% in 0.25% steps, amounts across each line's value and every boundary, on 104 lines, under caps of 5%, 15% and none | each outcome as recomputed in exact integers, and every accepted discount between zero and the line's value under all four rounding rules | **pass**: 0 wrong outcomes (03) |
| 11 | Eighteen planted defects | each caught | **pass** (04): all eighteen caught, worktree restored |

## Notes

- **A first run at `ad46290` let one planted defect survive:** dropping the cart's currency check. The test's 1.00 USD was below the line's 2,500 fils, so a later step reported the same mismatch. `a64942d` tests fifty dollars instead, 5,000 minor units, which without the check would be refused as exceeding the line: a cent weighed as a fil. All eighteen are caught at this head, and every result here is from `a64942d`.

## Not validated, and why

- **The caps in the shipped grid:** `role_capability.limit_json`, the shift lead's capped cell, and reading the cap back. This step's second half adds all three.
- **The basket discount** (1.4.6), **pricing the cart** (1.4.9), and **the `cart_discount_line` command** (1.4.11), with its check of the approver's own cap: none exists yet.
