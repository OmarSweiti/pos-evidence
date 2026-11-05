# Validation: 1.8.4's journal half, the checkout journal

**Commit validated:** `26f2876`, the pull request's head. It was checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **`pos-domain` gains `InFlightSale`**, the record of a sale being paid for:
  - its `Tendering` or `Finalizing` state;
  - any card operation outstanding;
  - the attempt's idempotency key;
  - when it last changed.
- **`pos-db` gains `CheckoutOperationRepository`** over migration 0005's register-local `checkout_operation`:
  - `begin` records a sale as payment begins;
  - `advance` records each step against the version it read;
  - `remove_on_finalize` deletes the row inside the finalize transaction;
  - `abandon` gives up a checkout still being paid for and returns it;
  - `get` and `in_flight` read rows back.
  - Each refusal has its own `DbError` variant.

No command calls the repository yet: 1.4.11's payment commands and 1.8.3's finalize will. The runtime is shown here through a harness linked against this head's `pos-db` and `pos-domain`. Each step runs it as a new process, on one register file per scenario. Each process opens the file with `pos_db::open` and calls the real repository.

**How it ran:**
- **The suites (01).**
  - The journal half's own suite, `cargo nextest run -p pos-db --test checkout_operation`, and the record's unit test in the domain.
  - Then every `pos-db` and `pos-domain` test, and clippy over both crates with `-D warnings`.
  - Then the test catalog and the frontier check.
  - The last block is `just pr`'s own gate at this head, in the main checkout.
- **The harness (02–03).**
  - Built `--offline` from the worktree's `Cargo.lock`, against a fresh synthetic register file per scenario.
  - Every step is its own process, and its exit status is recorded beside its output.
  - A step that dies calls `abort()`: no destructor runs and nothing commits.
  - The finalize is the harness's, written as 1.8.3 will write it, using:
    - `SaleRepository::prepare` and `insert_complete`;
    - the sale's stock events, its first receipt print job and its audit row;
    - its delivery envelope;
    - and `remove_on_finalize`, all in one transaction.
  - The database key is a synthetic test key passed to `pos_db::open`.
- **The planted defects (04).**
  - 17 of them, each applied to `repo/checkout_operation.rs` in the worktree.
  - Each was run against the journal suite, then restored with git.
  - No migration was mutated.

Data is synthetic throughout:
- an invented organisation and store;
- register `REG01` with an open shift on 2026-08-25;
- olive oil 1L at 1.250 JOD and bread at 0.350 JOD;
- sales A, B and C, and attempt keys K1 to K9.

The damaged-row case writes a buyer's name into a snapshot inside a transaction that is rolled back. The board builder refuses to render any board holding that name, the database key, a local path, or an unexpected acceptance.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db --test checkout_operation`; the record's unit test | 9 pass; 1 passes | **pass**: 9 of 9; `an_in_flight_sale_is_identified_by_its_own_cart` passes (01) |
| 2 | Every `pos-db` test (including the source-reading `repository_transaction_api`), every `pos-domain` test, clippy, the catalog, the frontier | all pass | **pass**: 326 of 326; 384 of 384; clippy clean; both checks reconcile, with 1.8.4 partial (01) |
| 3 | `just pr`'s gate at this head | lint, tests, web build, guards and the secret scan pass | **pass**: 900 passed, 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; no secret found (01) |
| 4 | Sale A, 2.850 JOD, begins payment under K1 | journalled at version 0, before any tender | **pass**: `tendering version 0 key K1`, 2.850 due; no completed sale (02) |
| 5 | A cash tender of 1.000; then a tender of 1.850 whose process dies before COMMIT | the first recorded; the restart finds the journal at the last committed step | **pass**: exit 134; the next process reads `version 1`, `tenders [cash 1.000]`, 1.850 due (02) |
| 6 | The 1.850 tender again; then a copy read at version 1 written again | the tender recorded; the stale copy refused | **pass**: `version 2`; `CheckoutStale`: "advanced to version 2 since this copy was read at version 1" (02) |
| 7 | The finalize begins | committed on its own as `finalizing`, before the finalize transaction | **pass**: `finalizing version 3 key K1`, 0.000 due, no completed sale (02) |
| 8 | The finalize, dying inside its transaction after the row's removal | nothing of the sale exists; the journal is still finalizing under K1 | **pass**: exit 134; `completed sales: 0`; `sale A finalizing version 3 key K1` (02) |
| 9 | The finalize replayed under K1 | the sale and the removal commit together | **pass**: `(no rows)`; one completed sale, receipt `REG01-000001`, 2.850 JOD, two stock events (02) |
| 10 | Sale C journalled again; sale A under C's key | refused by name | **pass**: `CheckoutAlreadyJournalled`; `CheckoutKeyReused` (03) |
| 11 | A tender, and an abandon, for C once its finalize has begun | both refused | **pass**: `CheckoutFinalizing` twice (03) |
| 12 | C's row removed under another key; B's removed though B never began its finalize | both refused | **pass**: `CheckoutKeyMismatch`; `CheckoutNotFinalizing` (03) |
| 13 | A tender for a sale never journalled; a card operation on B | both refused | **pass**: `CheckoutNotJournalled`; `CheckoutCardOpNotJournalled`, "the terminal is Phase 2's" (03) |
| 14 | B's snapshot damaged with a buyer's name, then read back | refused naming the sale, never quoting the snapshot | **pass**: `StoredCheckoutInvalid`: "checkout of sale B's snapshot: not a sale this build reads"; the name appears nowhere (03) |
| 15 | B abandoned while being paid for; then B begun again under K4 | what was journalled comes back; B begins a new attempt | **pass**: `version 1 key K2 total 0.350 tenders [cash 0.200]`; then `tendering version 0 key K4`, with C untouched (03) |
| 16 | 17 planted defects | each turns the test that names its rule red | **pass**: 17 of 17 caught; every mutant compiled, and the worktree was restored with no changed file (04) |

## Not validated, and why

- **A command or screen driving the journal.** No command calls the repository yet. 1.4.11's payment commands call `begin`, `advance` and `abandon`, and 1.8.3's finalize calls `remove_on_finalize`. The app was not launched. The repository is shown through the functions those commands will call.
- **The finalize is the harness's.** The real finalize command is 1.8.3's. The harness writes the sale through the same repository calls the shared `sale_world` test fixture uses, and removes the row inside that transaction.
- **Start-up recovery.** `resume_in_flight` and the start-up pass that reads `in_flight` are 1.8.4's recovery half, which follows 1.8.3. Step 9's replay is the harness reading the journal, not the shell's recovery.
- **Card operations.** Phase 1 has no card terminal, so recording one is refused (step 13) and a row holding one is not read. The two card tests named by 1.8.4 wait for the recovery half's decision.
- **Opening with the key from the OS credential store.** The harness passes a synthetic key to `pos_db::open`, so the credential store is never touched.
