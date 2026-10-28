# Validation: microsteps 1.4.1 and 1.4.2, the cart's first types

**Commit validated:** `65e4be2`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- `cart::open`, `Cart`'s `Debug` and serde forms, and `CartError` are pure: no database, device, emulator or screen is involved.
- A throwaway harness, not in the repository, called the pull request's own functions and printed what they returned. Every id, name and amount is synthetic; the buyer's TIN is a patterned placeholder.
- Each planted defect changed one statement of `crates/pos-domain/src/cart.rs` in the worktree and was restored. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain cart::tests::` (the `Done when` this PR gives both steps, since the plan had none) | 7/7 | **pass** (01) |
| 2 | The domain crate, then the workspace | no failure | **pass**: 261/261; 656/656 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | `cart::open` with a live and a training context | empty, no customer or buyer, `Domestic`, the shift and moment kept, training as given, `Building` | **pass** (02) |
| 4 | A parked cart with a B2B buyer, rendered with `{:?}` | the buyer's name shown only as `[redacted]`, the TIN visible | **pass** (02) |
| 5 | The same cart serialized and read back | equal, and the buyer's name kept on the wire for the receipt | **pass** (02) |
| 6 | Every `CartError` variant of `ref/domain-api.md` §6.3, constructed and rendered | 17 variants, each message carrying its data; tax and money errors passed through | **pass** (03) |
| 7 | Nine planted defects | each caught | **pass** (04): all nine caught, worktree restored |

## Found and fixed while preparing this validation

- **The TIN check was vacuous.** At `ad26b53` the planted defect "the buyer's TIN is hidden from Debug output too" survived. The fixture's TIN was `000000000`, and every all-zero UUID printed beside it contains that string, so the check passed without the TIN. `65e4be2` gives the fixture a TIN no id contains, and the defect is caught (04).

## Not validated, and why

- **Every transition after `open`**: adding and changing lines, age confirmation, park and resume, discounts, overrides, tendering, pricing. Those are 1.4.3 to 1.4.9.
- **The later states and fields**: they arrive with their steps, as recorded in `ref/domain-api.md` §6, including the module cycle 1.4.8 must break.
