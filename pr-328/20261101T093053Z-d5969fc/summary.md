# Validation: microstep 1.4.10, every money-reversing transition returns its audit intent

**Commit validated:** `d5969fc`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- **A pure change:** the transitions are pure functions. No database, device, emulator or screen is involved.
- **A throwaway harness**, not in the repository, ran the pull request's own transitions on carts rung up through its own `add_line`, under a synthetic 16% tax band, and printed the intent each one returned. It also read the cart and tender tables of `ref/ipc-contract.md` §3 from the pull request's own copy and set each Phase-1 command's audited column beside what its transition returned when run. It ran as a release build. Every id, name, reason code and amount is synthetic.
- **Planted defects:** each one changed one place in `crates/pos-domain/src/cart.rs` in the worktree, ran the cart and pricing suites, and was restored, proptest seed files included. Where the compiler refused the change, the runner shows which lines it refused inside the transition list. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain cart::tests::` (this step's `Done when`) | every test passes, this step's two among them | **pass** (01) |
| 2 | The domain crate, the workspace | no failure | **pass**: 349/349; 748/748 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | Every cart property alone, release build, `PROPTEST_CASES=65536` | pass | **pass**: 11 of 11 (01) |
| 4 | A cash sale of 5.000 taken apart by its cashier: the cash taken back, back to building, both lines voided | every step that removes money or a line leaves a record: a `tender.remove` row, then two `line.void` rows, all by the cashier | **pass** (02) |
| 5 | A card tender with a provider reference taken back; a tender not on the sale; the owner asking for `sale.create` | the record carries the reference and what is still owed; the unknown tender is refused with no record; the owner is denied | **pass** (02) |
| 6 | Every Phase-1 cart and tender command in the IPC contract, against what its transition returns | the contract's audited column and the domain agree | **pass**, with one known difference: 15 agree. `sale_finalize` is marked audited and `complete` returns no intent, which is recorded as left to 1.8.3 (02) |
| 7 | Twelve planted defects | each caught | **pass** (03): ten by a failing test, two refused by the compiler inside the transition list, the worktree restored |

## Notes

- **The one difference in step 6 is deliberate and recorded.** `complete` returns no intent, because the completed sale is itself the record and completing reverses no money. Whatever the finalize command's audit row should record beside the sale, such as the drawer opening on a cash sale, belongs to 1.8.3, and a note there says so. The pull request names this under "Read before approving".
- **A new public transition left out of the list is caught when the test runs, not at compile time.** The test reads the module's own source. A transition already in the list that changes its duty is refused by the compiler, as the two compile-time defects show.
- **Five Phase-1 rows were not run, each for a stated reason:**
  - two are reads;
  - `cart_add_scan` is `add_line` again;
  - the department sale is 1.4.12's;
  - the training toggle is no cart transition.

## Not validated, and why

- **The commands that write these intents** (1.4.11 and the tender commands) and **the finalize transaction** (1.8.3): none exists yet.
