# Validation — microstep 1.3.3, `compute_line_tax`, exclusive mode

**Commit validated:** `38947cd`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- `compute_line_tax` is a pure function: no database, device, emulator or screen is involved.
- A throwaway harness, not in the repository, called the pull request's own function and printed what it returned.
- Every component code is synthetic (`SYNTH_*`). 16% and 5% are arithmetic vectors, not seeded rates: 16% is `ref/tax-jordan.md` §4's worked example, and the real rate pack is 1.3.7's.
- Each planted defect changed or deleted one statement in the worktree and was restored; the run ends with "restored: 0 changed files".

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain tax::tests::` (the Done when) | 17/17, the two named tests and the new property included | **pass** (01) |
| 2 | The domain crate, then the workspace | no failure | **pass**: 227/227; 604/604 with the 4 existing `#[ignore]` tests skipped (01). See the note on "1 leaky" below. |
| 3 | Exclusive 16%, under each of the four rounding rules | tax = net × rate rounded once by the rule passed; gross = net + tax exactly; net 1 078 gives tax 172 and gross 1 250 | **pass** (02) |
| 4 | Exclusive 5%, where the exact tax lands on half a fil | the four rules give their own answers (0.5 → 1, 0, 0, 1) | **pass** (02) |
| 5 | The result as it would be carried on the sale | the net as the component's base, the tax as its amount and total, no supply reason | **pass** (02) |
| 6 | Every destination × every reason × with and without evidence | only a domestic supply with no reason is computed; every other destination is `SupplyDestinationRefused`; a domestic supply claiming a reason is `ZeroRatingReasonRefused` | **pass** (03) |
| 7 | Non-domestic supplies in every store profile and both price modes | all refused the same way | **pass**: 32 of 32 (03) |
| 8 | Each case a later step computes | refused as `NotComputedYet`, naming the case: inclusive (1.3.4); no, two, fixed or tax-on-tax components (1.3.5); unregistered store (1.3.8) | **pass** (04) |
| 9 | Zone stores, a negative rate, 0%, and amounts past the `i64` range | zone stores charged the same way; negative refused; 0% charges nothing; overflow refused | **pass** (04) |
| 10 | Twelve planted defects | each caught | **pass** (05): all twelve caught, worktree restored |

## Notes

- **"1 leaky" in step 2.** On the first run of the domain crate, nextest marked one passing test "leaky". That means a child process kept the test's output open for a moment after it finished; it is a timing note, not a failure. Two reruns at the same head showed 227/227 with no note. The run did not record which test it was.
- **A domestic supply that carries only an evidence reference, with no reason, is computed normally** (03, the second line). The reference has no effect on the tax and isn't copied onto the result.

## Not validated, and why

- **Tax-inclusive prices and the rounding-policy gate:** 1.3.4.
- **Several components, fixed amounts, tax on tax:** 1.3.5. **The receipt tax summary:** 1.3.6. **The real rate pack:** 1.3.7. **The unregistered store:** 1.3.8.
- **A real sale:** nothing calls `compute_line_tax` until the cart (group 1.4).
