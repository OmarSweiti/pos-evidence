# Validation: microsteps 1.4.8 and 1.5.2, paying for a sale

**Commit validated:** `895510c`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- **A pure change:** the transitions are pure functions. No database, device, emulator or screen is involved.
- **A throwaway harness**, not in the repository, rang carts up through the pull request's own `add_line` and paid for them with its own transitions, under a synthetic 16% tax band with prices including tax. Its sweep compares the transitions with a model of the payment rules, written separately in the harness from 1.4.8's entry: what is owed, what each tender is asked, which tender settles the sale, what it rounds, and the change. The model's rounding to the coin step is written from the three directions' definitions. The harness ran as a release build. Every id, name, reason code and amount is synthetic.
- **Planted defects:** each one changed one place in the payment transitions in `crates/pos-domain/src/cart.rs` in the worktree, ran the cart and pricing suites, and was restored, proptest seed files included. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain cart::tests:: && cargo nextest run -p pos-domain --test cart_ui` (both steps' `Done when`) | every test passes, the sixteen this PR adds among them, and the compile-fail lane holds | **pass** (01) |
| 2 | The domain crate, the workspace | no failure | **pass**: 347/347; 746/746 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | The three properties alone, release build, `PROPTEST_CASES=65536` | pass | **pass** (01) |
| 4 | A tobacco pack, unconfirmed, then confirmed (1.4.3b's second half) | payment refused until an intent names the confirming operator | **pass** (02) |
| 5 | 1.247 paid 0.500 cash, 0.124 card, then cash; and 1.5.3's 0.624 card then cash | only the final cash tender rounded: 0.623 asked as 0.620, −0.003; nothing taken after | **pass** (02) |
| 6 | Every kind asked 1.248, then 1.247, against 1.247; cash 2.000 | cash asked 1.250 at the qirsh; every other kind refused above 1.247 and settled at it; 0.750 change from cash | **pass** (02) |
| 7 | 1.241 owed, rounded up: cash 1.241, 1.245, 1.249, then 1.250 | the first three refused as `Underpaid`, 1.250 settles at +0.009 | **pass** (02) |
| 8 | 0.004 left: card 0.000, then cash 0.000; 0.007 left: cash 0.000 | cash of nothing settles at −0.004; nothing refused where a qirsh is owed | **pass** (02) |
| 9 | An unknown code, a tender not collected in each state, one taken twice, dollars, below nothing, past what can be counted | refused by name | **pass** (02) |
| 10 | Removing a card, a cash tender, an unknown one, and one before a final cash tender of nothing | the rest taken again in order; the last refused until the empty one goes first | **pass** (02) |
| 11 | Back to building, with and without a tender | refused only while a tender is taken | **pass** (02) |
| 12 | Finalizing unpaid, an unpaid sale read back from storage as finalizing, then paid and completed | both refused as `Underpaid`; completed with its receipt number, rounding and 0.250 change; stored form reads back byte for byte | **pass** (02) |
| 13 | A basket discounted by all it is worth (E.18) | settled with no tender, refuses one, completes | **pass** (02) |
| 14 | A void from every state, and a cashier asking for `sale.void` | voided while being built, parked or paid for with no tender, audited with the stage; refused otherwise; the cashier denied | **pass** (02) |
| 15 | Every sale of one line from 0.000 to 0.130 and six larger totals, under five coin steps and three directions, paid by two tenders of three kinds and amounts around every boundary, settled by card, each tender removed in turn, then completed | 0 differing from the model | **pass**: 16,847,469 sales, 33,694,938 tenders asked and 33,362,466 removals compared, 0 differing (03) |
| 16 | Thirty-one planted defects | each caught | **pass** (04): all thirty-one caught, worktree restored |

## Notes

- **A property's discards aborted at a raised case count, and were fixed.** At `8989bd5` the two 1.5.2 properties discarded a cart with no line through `prop_assume!`. At 65,536 cases the discards passed proptest's limit of 1,024 and the run aborted, though CI's 4,096 cases stayed under it. `895510c` substitutes one item for such a cart, and this evidence is for that head. All thirty-one planted defects were also caught at `8989bd5`, before the pull request was opened, and no test needed adding.
- **The sweep is not vacuous.** With one defect planted (cash between the remainder and its rounding taken as partial), the same sweep counted 512,300 differences, the first being 0.001 cash taken against 0.001 owed under a five-fil step rounded up.
- **1.247 in cash against 1.247 owed is refused** when the qirsh rounds to the nearest. Cash is asked 1.250, and 1.247 cannot be paid in coin.

## Not validated, and why

- **The commands** (`cart_confirm_age` and the checkout commands, 1.4.11), **persisting a sale in progress** (1.8.4) and **the finalize transaction** (1.8.3): none exists yet.
- **Pending tenders** (CliQ, Phase 2): refused here by design.
