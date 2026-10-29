# Validation: microstep 1.4.4, parking a cart and resuming it

**Commit validated:** `3971782`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- `park`, `resume` and `ParkedCart` are pure functions and a type: no database, device, emulator or screen is involved. The register's stored copy of a parked cart is 1.8.2b's, so the "restart" here is the stored form being written and read back.
- A throwaway harness, not in the repository, called the pull request's own functions and printed what they returned. It ran as a release build. Every id, name, code and amount is synthetic, and the buyer's TIN is a patterned placeholder.
- Each planted defect changed `crates/pos-domain/src/cart.rs` in the worktree, ran the cart suite and the compile-fail lane, and was restored, proptest seed files included. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain cart::tests::` and `--test cart_ui` (the `Done when` this PR gives 1.4.4, since the plan had none) | 25/25 and 2/2 | **pass** (01) |
| 2 | The domain crate, then the workspace | no failure | **pass**: 281/281; 676/676 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | `prop_park_resume_roundtrip_is_identity` alone, release build, `PROPTEST_CASES=65536` (16 times the crate's count) | pass | **pass** (01) |
| 4 | A cart of two teas, 1.250 kg of cheese, a tobacco pack still waiting for its age check and a business buyer, parked | the parked cart unchanged; its stored form byte for byte the cart's; the buyer's name absent from its `Debug` rendering | **pass** (02) |
| 5 | The stored sale read back, then resumed at the same till | read back equal to what was parked; resumed exactly the cart that was parked; the age check still refuses | **pass** (02) |
| 6 | Resumed after a shift handover: the same register, another shift and cashier, eight hours later | only `shift_id` and `cashier_id` differ; lines, prices, buyer and opening moment unchanged; the age check still refuses | **pass** (02) |
| 7 | An empty cart parked; resumed at another register, at a training till, a training cart at a live till, a till in dollars | each refused by name, with its message | **pass** (02) |
| 8 | Every till a live and a training cart could be offered to: register × training mode × currency × shift × cashier, 96 tills | each outcome as the PR says: 48 another register, 24 the other mode, 16 another currency refused; 8 resumed in the till's shift by its cashier | **pass**: 0 wrong outcomes (03) |
| 9 | Twelve planted defects, including a `From<Cart>` that lets a cart that was never parked be resumed | each caught | **pass** (04): all twelve caught, the last by the compile-fail lane, worktree restored |

## Notes

- The pull request was opened at `5cb6206`. `3971782` corrects one sentence of `resume`'s documentation: a sale resumed in the other training mode is counted wrongly either way round, not always left out. No code changed between the two, and this evidence is from `3971782`.

## Not validated, and why

- **The parked-cart table, its restart handling and its expiry** (1.8.2b), and **the `cart_park` and `cart_resume` commands** with their `sale.park` and `sale.resume` permission checks: none exists yet.
- **Paying for a resumed cart** (1.4.8): tendering is not built yet.
