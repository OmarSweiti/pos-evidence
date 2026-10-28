# Validation: microstep 1.3.8, a store not registered for GST charges none

**Commit validated:** `96f2d26`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- `compute_line_tax`, `summarize_tax` and `ReceiptModel::validate` are pure functions: no database, device, emulator or screen is involved.
- A throwaway harness, not in the repository, called the pull request's own functions and printed what they returned. It ran as a release build, for the sweep.
- Every component code is synthetic (`SYNTH_*`), every rate and amount an arithmetic vector, and the evidence receipt's merchant is invented. No rate pack is seeded.
- Each planted defect changed one statement of `crates/pos-domain/src/tax.rs` or `receipt.rs` in the worktree and was restored. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain unregistered_` (the `Done when`) | 4/4 | **pass** (01) |
| 2 | The tax and receipt suites, the domain crate, the workspace | no failure | **pass**: 60/60; 254/254; 649/649 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | One 10.000 JOD line for a registered and an unregistered store, handed 16%, a stacked special tax, a malformed loop, and nothing | the registered store charges or refuses as before; the unregistered one is untaxed every time, net and gross the price, no component | **pass** (02) |
| 4 | An unregistered store's sale abroad | refused by destination, as for every store | **pass** (02) |
| 5 | Its receipt's tax summary; an untaxed line beside a taxed one; an untaxed line claiming a fil of tax | no rows; refused as `NoComponents`; refused as `LineTaxUnbalanced` | **pass** (02) |
| 6 | The `Done when`'s independent fixtures: unregistered and registered GST, each with an invoice block and without | four valid sale documents, each carrying exactly the invoice block it was given | **pass** (03) |
| 7 | Sweep: 2 000 000 random lines for an unregistered store from a fixed seed, any components, price, quantity, mode and rule | every line untaxed, every summary empty, every sale abroad refused | **pass**: 0 mismatches (04) |
| 8 | Eight planted defects | each caught | **pass** (05): all eight caught, worktree restored |

## Notes

- **One timing report, no failure.** In step 1's run, nextest reports one of the four passing tests as "leaky": a handle still open just after it ended. The other suites, run right after it at the same head, report none, and no outcome changed.
- **Three findings outside this step's code**, recorded in the phase file at 1.8.3 and 1.8.5b:
  - `0005` refuses a completed line with no tax component, so an unregistered store cannot finish a sale until a forward migration allows it.
  - `StoreTaxProfile` folds the jurisdiction and the GST registration that `store` keeps in two columns.
  - `store.gst_profile` defaults to `'unregistered'` while merchant decision 6.2b assumes the merchant is registered.

## Not validated, and why

- **An unregistered store's sale through the database**: blocked by `0005`'s completion gate until 1.8.3's migration.
- **Reading the store's columns into the engine's profile** (1.8.3) and **setting GST registration at provisioning** (1.8.5b).
