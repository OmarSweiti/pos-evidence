# Validation: microstep 1.3.6, the receipt tax summary

**Commit validated:** `f190320`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- `summarize_tax` and `compute_line_tax` are pure functions: no database, device, emulator or screen is involved.
- A throwaway harness, not in the repository, called the pull request's own functions and printed what they returned. It ran as a release build, for the sweep.
- Every component code is synthetic (`SYNTH_*`), and every rate and amount is an arithmetic vector. A zero-rated or exempt `SYNTH_GST` is a treatment the engine is handed, not a claim about which Jordanian supplies are either. No rate pack is seeded.
- Each planted defect changed one statement of `crates/pos-domain/src/tax.rs` in the worktree and was restored. The run ends with "restored: 0 changed files", and the harness was built before it started.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain tax::tests::` (the `Done when` suite) | 41/41: 33 existing, 8 new | **pass** (01) |
| 2 | The domain crate, then the workspace | no failure | **pass**: 251/251; 635/635 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | The `Done when`: two zero-rated lines, one exempt, one at 16% and one at 4% on one receipt, with prices before tax and then tax-included | a zero-rated row and an exempt row, each with 0.000 tax, never one row; with one component a line, the rows add up to the receipt in tax, net and gross | **pass** (02) |
| 4 | A reference line beside a line differing in one fact: code, treatment, percentage, fixed amount, having a fixed amount, zero-rating reason; then beside itself | two rows each time; one row of exactly both lines beside itself | **pass** (03) |
| 5 | A fixed special tax with the general tax on the net plus it, beside the general tax alone | one general-tax row for both lines, its net the base 16% was charged on (12.500, tax 2.000); the taxes add up to the receipt's, the nets by design do not | **pass** (04) |
| 6 | A line changed after the engine computed it: a component, the tax total, the gross, the net, every component removed | refused by position as `LineTaxUnbalanced(1)`, or `NoComponents` | **pass** (05) |
| 7 | Two currencies; a fixed amount in another currency; nets, taxes and grosses each overflowing on their own; no lines | refused by name; no lines gives no rows | **pass** (05) |
| 8 | `compute_line_tax`: zero-rated and exempt components at 0%, a fixed amount of nothing, and charging a rate or an amount, in both price modes | nothing charged when nothing is asked; `UntaxedTreatmentCharges` whenever a rate or an amount is, even one ppm or one fil; a reduced rate is charged | **pass** (05) |
| 9 | Sweep: 2 000 000 random receipts from a fixed seed, held to a grouping the harness computes on its own, re-summarized in reverse and shuffled order | every check 0 mismatches; every zero-rated or exempt component given a rate or amount refused | **pass**: 9 718 383 lines, 12 540 866 components, 0 mismatches; 396 386 of 396 386 refused (06) |
| 10 | Nineteen planted defects | each caught | **pass** (07): all nineteen caught, worktree restored |

## Found and fixed while preparing this validation

A first run of fifteen planted defects, at `2a119cc`, caught thirteen. Commit `bb2df2c` addresses both survivors, and all of this evidence was re-run at the head.
- **Dropping the check of each line's currency changed nothing.** Summing a line's taxes in the receipt's currency already refuses a line in another one, so the check was redundant and was removed. The test now gives the foreign line a code of its own, so only that per-line sum can stop it.
- **Letting a row's net wrap on overflow went uncaught**, because the test overflowed nets and grosses together and the gross check refused first. The test now overflows the nets, the taxes and the grosses one at a time, and each wrap is caught (07).

The same commit adds the engine refusal `UntaxedTreatmentCharges`. Without it, "exempt and zero produce zero tax" held only for the 0% rates the property was handed.

## Notes

- The zero-rating reason is caught by an example only (`summary_rows_are_keyed_by_every_rate_defining_fact`). The engine refuses every reason until the accountant names the zero-rated lines (`ref/merchant-decisions.md` 6.14), so no property can draw a line that carries one, and that example builds its line by hand.
- The planted-defect page leaves out the assertion values the failing tests printed. The run's filter for each summary line also matched the type name `TaxSummaryRow`, and those lines are not outcomes. Every FAIL and Summary line is kept.

## Not validated, and why

- **How a row is stored in `sale_tax_summary`.** It keeps a compound component as two rows and has a `taxable_base_minor` beside `net_minor`; that mapping is 1.8.3's.
- **Printing the summary** (group 1.7), **the real rate pack** (1.3.7) and **unregistered stores** (1.3.8).
