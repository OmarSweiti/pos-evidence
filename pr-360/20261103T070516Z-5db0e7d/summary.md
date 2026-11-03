# Validation: 1.8.2, `SaleRepository` and migration 0009

**Commit validated:** `5db0e7d`, the pull request's head, checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **Migration `0009`** adds the columns for what a completed sale captured. Its completion gate refuses a sale without them.
- **`pos_db::repo::sale::SaleRepository`** writes a completed sale with `prepare` and then `insert_complete`, and reads it back with `by_id`, `by_receipt_number` and `for_business_date`.
- **The domain:** `CompletedSale::restore` re-checks a stored sale before it's handed on.

No screen calls the repository yet (finalize is 1.8.3), so the runtime is shown through the repository itself on synthetic register files.

**How it ran:**
- **The suites.** The `Done when`, the migration's suite, the domain's stored-form tests (the property again at `PROPTEST_CASES=20000`), every `pos-db` test, clippy, both schema verifiers, and the whole workspace (01).
- **The harness.** A harness linked against this head's `pos-db` and `pos-domain`, built `--offline` from the worktree's `Cargo.lock`. It completes a synthetic sale through the domain and writes it the way 1.8.3's finalize will, in one transaction:
  - the sale prepared;
  - one envelope naming every fact;
  - the sale's rows, then its stock events, receipt and print job, audit rows and the approval consumed against it;
  - then the completion.

  It then reads the sale back and attempts each refusal (02, 03, 04).
- **The planted defects.** 19, each applied to the worktree, tested and restored. No migration was mutated (05).

Data is synthetic throughout: an invented store, register, products, buyer and card. The buyer's name is held but appears on no board.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db --test sale_repository_api` | 11 pass, the compile-fail fixture among them | **pass**: 11 of 11 (01) |
| 2 | `migration_0009_sale_capture`, the domain's `restore` tests, `pos-db`, clippy, `verify-schema`, `verify-pg-migrations --mapping-only`, the workspace | all pass | **pass**: 4 of 4; 2 of 2, and the property at 20,000 cases; 282 of 282; clippy clean; both verifiers conform; 817 passed, 4 skipped (the existing `#[ignore]` ones) (01) |
| 3 | A sale with every optional fact, written in the finalize's order and read back by `by_id` | the same `CompletedSale`, field for field, with its store, policy, commit, original receipt, approvals and tender states | **pass**: `read back == completed: true`. Both read paths (`by_receipt_number`, `for_business_date`) return the same record, and no `Debug` carries the buyer's name (02) |
| 4 | What travels and what is stored | 17 sale facts, parents first, each in the envelope with its payload; `0009`'s columns holding what the cart captured | **pass**: the label line stores unit `package` beside its product's weight unit; only the settling tender gives change (03) |
| 5 | `insert_complete` against an envelope missing a fact, an envelope carrying another payload, and a finalize with no original receipt | each refused by name, no row left | **pass**: `SaleEnvelopeMissing`, `SaleEnvelopeMismatch`, and `0005`'s durable-outputs gate; 0 sale rows left; the same sale written whole then completes (04) |
| 6 | `prepare` with a department line, a ceiling in another currency, and an exclusive-priced sale told it was inclusive | each refused before anything is written | **pass**: the department line names #330; the other two name what would not read back. Told its own mode, the sale prepares (04) |
| 7 | `0009`'s gate, each captured fact withheld once | each refused; nothing withheld completes | **pass**: 6 refusals, then `COMPLETED` (04) |
| 8 | The compile-fail fixture | no `update`, `delete`, `void` or `complete` on the repository | **pass**: four `E0599` errors (04) |
| 9 | 19 planted defects | 18 turn the test that names their rule red; M19 survives, as predicted | **pass**: 18 of 18 caught; M19 (no `AND status = 'parked'`) survived. Every mutant compiled, and the worktree was restored with no changed file (05) |

`just pre-push` was run by `just pr` before the pull request was opened, and passed: lint, tests, the web build, guards and the secret scan.

## Not validated, and why

- **A register app completing a sale on screen.** Finalize, which calls this repository, is 1.8.3's. The repository is shown here through the same calls 1.8.3 will make.
- **The Postgres engine pass.** The mirror creates nothing. `verify-pg-migrations.py --mapping-only` ran here; CI's `rust` job runs the engine pass.
- **The required `supply-chain` check on this pull request is red, for a reason this change doesn't touch.** `async-executor` 1.14.0 in the lock was yanked on 2 November, and `development` fails the same check. A separate pull request moves the lock to 1.13.3.
