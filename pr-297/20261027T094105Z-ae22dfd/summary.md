# Validation — microstep 1.3.5: several tax components on a line

**Commit validated:** `ae22dfd`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- `compute_line_tax` and `resolve_components` are pure functions: no database, device, emulator or screen is involved.
- A throwaway harness, not in the repository, called the pull request's own functions and printed what they returned. It ran as a release build, for the sweep.
- Every component code is synthetic (`SYNTH_*`), and every rate and amount is an arithmetic vector. `SYNTH_SST` and `SYNTH_GST` follow the default arrangement of the Special Sales Tax `⚠️ OPEN` item in `ref/domain-api.md` §5; no such rule is seeded.
- Each planted defect changed one statement of `crates/pos-domain/src/tax.rs` in the worktree and was restored. The run ends with "restored: 0 changed files", and the harness was built only after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain tax::tests::` (the `Done when`) | 33/33: 23 existing, 10 new | **pass** (01) |
| 2 | The domain crate, then the workspace | no failure | **pass**: 243/243; 627/627 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | A fixed special tax, an ad-valorem one on it, and a general tax on both, listed in reverse | applied by sequence; bases 10000 / 10500 / 11550; tax 3398; the listing order changes nothing | **pass** (02) |
| 4 | Both shapes in one component, then general tax on it | 1500 on 10000, then 1840 on 11500 | **pass** (02) |
| 5 | 1.5 units at 333 fils a unit, under each rounding rule | rounded once (499.5 → 499 under floor, 500 otherwise), and the taxes above it charged on the rounded amount | **pass** (02) |
| 6 | General tax on the net plus special tax, against general tax on the net alone | 176 on 1100, against 160 on 1000 | **pass** (02) |
| 7 | A tax-inclusive price on one fixed or combined component | net rounded once, tax the residual, net + tax = price under every rule | **pass** (03) |
| 8 | A tax-inclusive price smaller than its fixed tax, and a price of zero | refused as `FixedTaxExceedsPrice` | **pass** (03) |
| 9 | A tax-inclusive price on two components | refused as `InclusiveSplitUndefined`; the same components charge an exclusive price | **pass** (03) |
| 10 | Malformed lists: a loop, a base not applied first, a base not on the line, none, a code twice, a negative or foreign-currency fixed amount | each refused by name, identically in both price modes | **pass** (04) |
| 11 | Rate packs for each store profile | a zone pack lacking a standard component is `ProfilePackIncomplete`; a complete one resolves its own rate; no rule anywhere is `NoRuleInEffect` | **pass** (05) |
| 12 | Sweep A: one fixed component, alone and beside 16%, three amounts and quantities, every price from 1 fil to 1000.000 JOD, all four rules | every net agrees with integer division; refused exactly where integer division gives a negative net | **pass**: 24 000 000 lines, 0 mismatches (06) |
| 13 | Sweep B: the three-component stack at 1 and 1.5 units, every net from 1 fil to 1000.000 JOD, all four rules | every component, base, tax and gross agrees with integer division | **pass**: 8 000 000 lines, 0 mismatches (06) |
| 14 | Sixteen planted defects | each caught | **pass** (07): all sixteen caught, worktree restored |

## Found and fixed while preparing this validation

The first run, at `b8d4ab1`, caught all sixteen defects, but showed two thin spots. Commit `ae22dfd` addresses them, and all of this evidence was re-run at that head.

- **Rounding a combined component's two parts separately** was caught only by a property, with no fixed example. Every example's parts were whole fils. The special-tax fixture now charges 10% of 1005 plus 333 fils on 1.5 units: exactly 600, where separate rounding gives 601. The defect is now caught by both (07).
- **`prop_inclusive_fixed_amount_extracts_exactly` almost never drew a price below its fixed tax**, because its prices came from the whole of `i64`. Half its prices now come from zero to a million dinars, so its refusal branch is exercised.

## Notes

- The defect "a zero price may carry a negative net" is caught by a fixed example (`inclusive_fixed_amounts_extract_exactly`) and by no property. It differs from the correct code only at a price of exactly zero, which a random draw almost never lands on, so the example is the right tool.

## Not validated, and why

- **Which base, order and per-unit or per-line treatment Jordan's Special Sales Tax requires**, and **how a tax-inclusive price divides between several components**: the `⚠️ OPEN` item in `ref/domain-api.md` §5, now owned by 1.3.7. No special-tax rule is seeded.
- **The receipt tax summary** (1.3.6), **the real rate pack** (1.3.7), **unregistered stores** (1.3.8).
