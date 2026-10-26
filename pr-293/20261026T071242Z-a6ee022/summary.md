# Validation — microstep 1.3.2, `resolve_components`

**Commit validated:** `a6ee022`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- `resolve_components` is a pure function: no database, device, emulator or screen is involved.
- A throwaway harness, not in the repository, called the pull request's own function on a synthetic rate pack at the instants that matter.
- Every component code and rate is synthetic (`SYNTH_*`, at rates no schedule uses), so nothing here claims a real Jordanian rate. The real rate pack is 1.3.7's.
- Each planted defect edited one line in the worktree and was restored; the run ends with "restored: 0 changed files".

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain tax::tests::` (the Done when) | 10/10, the boundary property included | **pass** (01) |
| 2 | The domain crate, then the workspace | no failure | **pass**: 220/220; 597/597 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | A rate that changes at midnight | the old rule up to 23:59:59.999, the new one from 00:00:00.000, nothing before the first rule, and the same answers in reverse order | **pass** (02) |
| 4 | Four store profiles on one pack | the scoped rule overrides the unscoped one; ASEZ gets only its own rate; profiles with no rules are refused, not given the mainland rate | **pass** (03) |
| 5 | An overlap, another category's rules, an empty pack | each refused by name; outside the overlap, either rule answers alone | **pass** (04) |
| 6 | Three components listed out of order | returned by sequence, then code; only the plain line-net percentage is inclusive-capable | **pass** (04) |
| 7 | Seven one-line defects | each caught | **pass** (05): all seven caught, worktree restored |

## Found and fixed while preparing this validation

- **A test that couldn't catch one defect.** `resolved_components_come_in_application_order` would have passed with the sort by sequence deleted, because its codes sorted the same way alphabetically. Commit `4d50e2c` makes the last-applied component's code sort first; the defect is now caught (05).
- **An unreadable refusal.** `NoRuleInEffect` printed its instant as raw milliseconds (`Timestamp(1767225599999)`). Commit `a6ee022` prints it as a date, `2025-12-31T23:59:59.999Z`, with a test that pins the text. All of this evidence was re-run at that head.

## Not validated, and why

- **Tax arithmetic, and the refusal of non-domestic supplies:** 1.3.3 onward. This step records the owner's decision (questionnaire row 6.14); 1.3.3 enforces it.
- **Naming an incomplete zone pack (`ProfilePackIncomplete`):** 1.3.5. **The unregistered short-circuit:** 1.3.8. Here both fail safe as "no rule in effect".
- **A real rate pack:** 1.3.7.
