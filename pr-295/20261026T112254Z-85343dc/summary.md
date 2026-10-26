# Validation — microstep 1.3.4, pure half: tax extracted from a tax-inclusive price

**Commit validated:** `85343dc`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- `compute_line_tax` is a pure function: no database, device, emulator or screen is involved.
- A throwaway harness, not in the repository, called the pull request's own function and printed what it returned. It ran as a release build, so its sweep could cover all four rounding rules.
- Every component code is synthetic (`SYNTH_*`), and every rate is an arithmetic vector, not a seeded rate. 16% is `ref/tax-jordan.md` §4's worked example; the real rate pack is 1.3.7's.
- Each planted defect changed or deleted one statement in the worktree and was restored; the run ends with "restored: 0 changed files".

**Scope:** this pull request is 1.3.4's pure half. The database half (`tax_policy.rs`) and the terminal half (`unapproved_tax_computation_policy_blocks_provisioning_and_finalize`, which needs files 1.8.3 and 1.8.5b create) aren't in it, so the `Done when`'s second and third commands weren't run. The step's `Full-step status:` marker says so.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain inclusive_extraction_is_exact_over_the_whole_fils_range` (the `Done when`'s pure command) | every gross 1..=1 000 000 at all seven vector rates agrees with integer division | **pass** (01) |
| 2 | `cargo nextest run -p pos-domain tax::tests::` (the pure half, as its `Full-step status:` names it) | 23/23 | **pass** (01) |
| 3 | The domain crate, then the workspace | no failure | **pass**: 233/233; 610/610 with the 4 existing `#[ignore]` tests skipped (01) |
| 4 | The worked example, 1250 at 16% | net 1078 + tax 172 = 1250 | **pass** (02) |
| 5 | Gross 4 at 16%, where the tax must be the remainder | net 3 + tax 1; a tax rounded on its own would be 0 | **pass** (02) |
| 6 | The 12.5-fil tie, 13 at 4% | 13 + 0 under half-away-from-zero; 12 + 1 under half-even | **pass** (02) |
| 7 | Every gross 1..=1 000 000, seven rates, all four rounding rules (wider than the test, which runs two) | every net agrees with integer division; every net + tax equals the gross | **pass**: 28 000 000 of 28 000 000, 0 mismatches (03) |
| 8 | Inclusive prices that must be refused | a non-extractable component, later-step shapes, a negative rate and an export are each refused by name | **pass** (04) |
| 9 | Gross at both ends of `i64` | extraction never overflows; net + tax equals the gross | **pass** (04) |
| 10 | Nine planted defects | each caught | **pass** (05): all nine caught, worktree restored |

## Found and fixed while preparing this validation

- **A defect only one test caught.** On the first run, at `6531a88`, the defect that rebuilds an inclusive line's gross from its net was caught by the property test alone, with no fixed example covering it. Commit `85343dc` adds gross 4 at 16% to `inclusive_16pct_extracts_exactly`: the smallest line where the remainder matters (net 3, tax 1, where a separately rounded tax would be 0). The defect is now caught by both (05). All of this evidence was re-run at that head.

## Notes

- One mutant run in the first pass, at `6531a88`, reported a passing test as "leaky": a child process held its output open for a moment after the test finished. It's a timing note, not a failure, and the re-run at this head showed none.

## Not validated, and why

- **1.3.4's database half:** the next pull request. **Its terminal half:** at 1.8.3 / 1.8.5b.
- **Which rounding rule Jordan requires:** the `⚠️ OPEN` item in `ref/schema.md` §0003. Until it's answered, no policy is approved and no sale can be finished; that's the plan's deliberate default.
- **Several components, fixed amounts, tax on tax:** 1.3.5. **The real rate pack:** 1.3.7.
