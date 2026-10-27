# Validation — microstep 1.3.4, database half: a store's approved tax policy, read into its rounding rule

**Commit validated:** `77ae858`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- `TaxPolicyRepository` reads a register database; no device, emulator or screen is involved.
- A throwaway harness, not in the repository, opened **fresh, encrypted register databases** at schema version 7 and seeded them with `pos-db`'s own `RegisteredChain` fixture. It then called the pull request's own repository and printed what it returned.
- Every id, store code and policy version is synthetic. The one approved policy is the test fixture's: no real policy is approved until the `⚠️ OPEN` rounding item in `ref/schema.md` §0003 is answered.
- The harness's databases were deleted after the run.
- Each planted defect changed one statement of `crates/pos-db/src/tax_policy.rs` in the worktree and was restored. The run ends with "restored: 0 changed files", and the harness was built only after it.

**Scope:** this pull request is 1.3.4's database half. The terminal half (`unapproved_tax_computation_policy_blocks_provisioning_and_finalize`) needs files that 1.8.3 and 1.8.5b create, so the `Done when`'s terminal command wasn't run. The step's `Full-step status:` marker says so.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db tax_policy::` (the `Done when`'s database command) | 7/7 | **pass** (01) |
| 2 | `cargo nextest run -p pos-domain tax::tests::` (the pure half, unchanged) | 23/23 | **pass** (01) |
| 3 | The `pos-db` crate, then the workspace | no failure | **pass**: 224/224; 617/617 with the 4 existing `#[ignore]` tests skipped (01) |
| 4 | The fixture's approved policy (`half_even`, a 5-fil step, `nearest`) is read | `HalfEven`, 5, `Nearest`, its version and approval time, and the id the store is pinned to | **pass** (02) |
| 5 | The rule that was read is handed to `compute_line_tax` | the 13-fil tie at 4% records 12 + 1 (half-even), not the provisional default's 13 + 0; the worked example is 1078 + 172 | **pass** (02) |
| 6 | The row's coin step and direction are handed to `compute_cash_rounding` | 1247 fils is asked as 1245, adjustment −2 | **pass** (02) |
| 7 | A store pinned to a draft policy, or to none, is set up or re-pinned | the database refuses all four; the read names the unsaved store as unknown, and the existing store keeps its approved policy | **pass** (03) |
| 8 | Each rule and direction is written in the schema's spelling, then in the serde wire spelling | the schema spelling is accepted and reads back; the wire spelling is refused by the CHECK | **pass**: 7 of 7 each way (04) |
| 9 | The trigger dropped, then the foreign key and CHECKs switched off, on a scratch database | no policy, an unapproved draft, a policy not held, an unknown rule, an unknown direction, a zero step, an unreadable approval time, a 15-byte id, and a store not held are each refused by name | **pass** (05) |
| 10 | Eleven planted defects | each caught | **pass** (06): all eleven caught, worktree restored |

## Found and fixed while preparing this validation

- Nothing in the pull request. One evidence board was re-rendered: its colouring had turned the word "accepted" green inside two defect titles, which read as an outcome. The board's text is unchanged.

## Not validated, and why

- **1.3.4's terminal half:** at 1.8.3 / 1.8.5b.
- **Which rounding rule Jordan requires:** the `⚠️ OPEN` item in `ref/schema.md` §0003. Until it's answered, no policy is approved and no sale can be finished; that's the plan's deliberate default, and step 7 shows it holding.
- **Where the coin step should live:** `0003` stores it on the policy row, while `ref/tax-jordan.md` §5 and merchant decisions 2.1–2.2 place it on `store`. The phase file records the discrepancy, and this pull request doesn't resolve it.
