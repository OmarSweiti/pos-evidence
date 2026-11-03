# Validation: 1.8.2b, the parked-cart repository

**Commit validated:** `48465c6`, the pull request's head, checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **`pos_db::repo::parked_cart::ParkedCartRepository`** keeps a parked basket in `parked_cart` from the moment it is set aside until the sale it becomes completes. It has `park`, `resume` (the claim), `save_active`, `repark`, `consume_on_finalize`, `reclaim_after_restart`, and the `parked` and `expired` lists.
- **The domain's `resume`** now takes the till cashier's `sale.resume` proof. It returns an audit intent when the basket was parked by another cashier.

No screen calls the repository yet: `cart_park`, `cart_resume` and `cart_list_parked` are 1.4.11's. The runtime is therefore shown through the repository itself, on synthetic register files.

**How it ran:**
- **The suites.** The `Done when`, the domain's park and resume tests (the round-trip property again at `PROPTEST_CASES=20000`), 1.8.2's suite on the shared fixture, every `pos-db` test, clippy and the whole workspace (01).
- **The harness.** A harness linked against this head's `pos-db` and `pos-domain`, built `--offline` from the worktree's `Cargo.lock`. Every step is its own process, run one after another on one register file. A step that dies calls `abort()`, so no destructor runs, nothing is rolled back by the program and nothing commits. Each process's exit status is recorded beside its output (02, 03, 04). A second fresh file holds the refusals (05).
- **The planted defects.** 19, each applied to the worktree, tested and restored. No migration was mutated (06).

Data is synthetic throughout: an invented store, register, cashiers (`FIX-1`, `C-2`), owner (`OWN-1`) and products. One damaged stored form carries an invented buyer's name, to show the refusal never quotes it; the name appears on no board.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db --test parked_cart` | 7 pass | **pass**: 7 of 7 (01) |
| 2 | The domain's park and resume tests, the property at 20,000 cases, `sale_repository_api`, `pos-db`, clippy, the workspace | all pass | **pass**: 6 of 6, and the property at 20,000 cases; 11 of 11; 289 of 289; clippy clean; 825 passed, 4 skipped (the existing `#[ignore]` ones) (01) |
| 3 | Two baskets parked, then a new process lists them | both offered, each exactly as parked, with its label | **pass**: cart-A "table 4" and cart-B, contents, parker and expiry as written (02) |
| 4 | A resume whose process dies before COMMIT | the basket is still parked, with no claim | **pass**: exit 134; the next process finds cart-A `parked`, claim `-` (02) |
| 5 | A resume that COMMITs, and whose process dies before the till hears | the next start hands the claim to the new session | **pass**: exit 134; `start 3` reclaims cart-A to S3 (02) |
| 6 | The dead session writes; the live one changes the basket and dies after COMMIT | the dead session is refused; the change survives the next start | **pass**: S2 refused as `ParkedCartNotClaimed`; S3's change committed; exit 134; `start 4` shows 3 × olive oil (02) |
| 7 | A finalize whose process dies inside its transaction, after removing the working cart | the basket is still there, and no sale exists | **pass**: exit 134; `start 5` finds cart-A claimed by S5, completed sales 0 (03) |
| 8 | The dead session's finalize, then the live session's | the dead one refused and no sale; the live one commits one sale and leaves no basket | **pass**: S4 refused as `ParkedCartNotClaimed`, sale exists `false`; S5 commits receipt `REG01-000001`, total 4.100; cart-A's row gone, completed sales 1 (03) |
| 9 | The owner, a proof that isn't the till's own, and C-2 taking over FIX-1's basket | no proof for the owner; the wrong proof refused; the takeover audited in the claim's transaction | **pass**: `OWN-1 lacks sale.resume`; `ResumeRefused` naming FIX-1 and C-2's till; `audit_log` actor C-2, `sale.resume`, parked_by FIX-1, line_count 1; cart-B now C-2's (04) |
| 10 | A basket parked on 25 August, resumed on the 26th | refused as expired, not offered, listed for the end-of-day report, kept | **pass**: `ParkedCartExpired`; `parked` none; `expired` lists cart-C "table 9"; its row is still there (04) |
| 11 | Every refusal on a fresh file | each refused by the error that names it, rows unchanged | **pass**: `ParkedCartAlreadyParked`, `ParkedCartShiftUnknown`, `ParkedCartNotFound`, two `ResumeRefused` (another register, training mode), `ParkedCartClaimed`, four `ParkedCartNotClaimed`, two `StoredParkedCartInvalid`; the refusal quotes the stored form: `false` (05) |
| 12 | 19 planted defects | each turns the test that names its rule red | **pass**: 19 of 19 caught. Every mutant compiled, and the worktree was restored with no changed file (06) |

`just pre-push` was run by `just pr` before the pull request was opened, and passed: lint, tests, the web build, guards and the secret scan.

## Not validated, and why

- **A register app parking and resuming a basket on screen.** The three IPC commands, and the shell's start-up call to `reclaim_after_restart`, are 1.4.11's. The repository is shown here through the calls 1.4.11 will make.
- **The resume's audit row from a command handler.** The harness writes the intent with its envelope as the caller must. 1.4.11's handler is where that happens in the app.
- **The required `protected-paths` check is red by design.** This pull request retires its `PLANNED` test name in the byte-pinned `scripts/check-test-catalog.py`, as every earlier retirement did.
