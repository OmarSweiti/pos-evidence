# Validation: microstep 1.4.3b, age confirmation (the domain half)

**Commit validated:** `ed20ae4`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- `confirm_age` and `check_age_confirmed` are pure functions: no database, device, emulator or screen is involved.
- A throwaway harness, not in the repository, called the pull request's own functions and printed what they returned. The cart holds a synthetic tea and a synthetic sealed pack of tobacco with a minimum age of 18. The confirming operator is a shift lead, deliberately not the cart's cashier.
- Each planted defect changed one statement of `crates/pos-domain/src/cart.rs` in the worktree and was restored, proptest seed files included. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain cart::tests::age_` (the plan's `Done when` filter) | passes; it runs four of this step's five tests, beside 1.4.3's age test | **pass**: 5/5 (01) |
| 2 | The cart suite, the domain crate, the workspace | no failure; the cart suite runs all five of this step's tests | **pass**: 21/21; 276/276; 671/671 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | A cart with tea and a blocked tobacco pack, before any answer | the tender gate refuses, naming the tobacco | **pass** (02) |
| 4 | The shift lead confirms | the line stays, confirmed; the gate opens; the intent is `age.confirm` on the line, naming the shift lead, the product and its minimum age, at the given moment, and nothing about the customer | **pass** (02) |
| 5 | The shift lead declines instead | the line is removed; the gate opens on what remains; the intent is `age.decline` | **pass** (02) |
| 6 | Answering the tea line, a line not in the cart, and the tobacco again after it was confirmed | each refused by name; the original cart's gate still refuses | **pass** (02) |
| 7 | The cart redrawn from its serialized form five times | the tobacco line is there, blocked, under the same id; confirming it by that id opens the gate | **pass** (02) |
| 8 | Eight planted defects | each caught | **pass** (03): all eight caught, worktree restored |

## Found and fixed while preparing this validation

- **The confirming operator was the cart's own cashier in every test**, so a defect that recorded the cashier instead of the person who answered could not have been told apart. `ed20ae4` makes the operator a shift lead, and that defect is caught (03).

## Not validated, and why

- **Reaching `Tendering`**: the second half of this step's `Done when`, which needs 1.4.8's `begin_tender`. Until then `check_age_confirmed`, the gate it will call, stands in, and 1.4.3b keeps its `Full-step status:` marker.
- **The `cart_confirm_age` command**, which lands in 1.4.11's new `commands/cart.rs`.
