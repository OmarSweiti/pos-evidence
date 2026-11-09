# Validation: #397, a sale audited before it completes still completes

**Commit validated:** `225f965`, the pull request's head. It was checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:** four audit intents the domain makes before a sale completes name the sale's basket, `parked_cart`, under the same id, instead of the `sale`:
- `tender.remove`;
- the takeover's `sale.resume` from `cart_resume`;
- `discount.manual` on the basket;
- `sale.department`.

Before, a sale with a tender taken back, or with its basket taken over from another cashier, could never complete. Migration `0005`'s `sale_commit_base_complete` requires every audit row naming the `sale` to travel in the sale's own commit, and those rows were already in commits of their own. `sale.void` still names the `sale`.

The runtime is shown through a harness linked against this head's `terminal`, `pos-db`, `pos-domain` and `pos-hardware`.
- **Each step is a new process.** It opens the register with `Register::open_with` on a file built by `pos_db::open`, as `run` opens it.
- **It drives the shipped commands:** sign-in and sign-out, the cart, `cart_resume`, the tenders, `tender_remove` and `sale_finalize`.
- **The receipt prints** to 1.7.12's emulated printer through 1.7.10's network driver, both started in the harness's process on `127.0.0.1`.

**How it ran:**
- **The suites (01):** run in the worktree:
  - the two regression tests, `cargo nextest run -p terminal still_completes`;
  - the four intents' own domain tests, and the void's;
  - every `pos-domain` and `terminal` test;
  - clippy over both crates with `-D warnings`.

  Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–03):**
  - Built `--offline` against the worktree, with a fresh synthetic register file per scenario.
  - Each step's exit status is recorded beside its output.
  - The board builder refuses an unexpected exit, a failed check, the fixture's PIN or a local path.
- **The planted defects (04):**
  - 6 of them, run at this head before the pull request's second gate.
  - Each was run against the test that names its rule, then restored with git.
  - The worktree was clean afterwards.
- **Builds ran without debug information** (`CARGO_PROFILE_DEV_DEBUG=0`).

**Data is synthetic throughout:**
- the reference chain's fixture merchant and its store;
- register `REG01`, with its shift open on 2026-11-05;
- cashiers C-100 and C-200, whose PIN is typed by the harness and printed nowhere;
- `SYN-` products.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal still_completes` | both regression tests pass | **pass**: 2 of 2 (01) |
| 2 | The four intents' own domain tests, and the void's | each expects its entity | **pass**: 5 of 5 (01) |
| 3 | Every `pos-domain` and `terminal` test; clippy | all pass; clean | **pass**: 397 of 397 and 107 of 107; clippy exit 0 (01) |
| 4 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `225f965`: 982 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog reconciles (01) |
| 5 | Oil and 101 g of cheese; 1.000 cash taken, then taken back; 5.000 settles it; finalize | the sale completes and prints; the removal's row names the basket; the sale's commit is whole | **pass**: `REG01-000001`, total charged 2.110, change 2.890, receipt `Printed`, drawer opened. `tender.remove` on `parked_cart` under the sale's id, in a commit of its own; `sale.finalize` on `sale`, in the sale's commit; one commit names the sale, 13 members, 13 delivery rows; no journal or basket row left (02) |
| 6 | C-100 builds the basket and signs out; in a new process C-200 takes it over, pays 2.110 and finalizes | the sale completes and prints; the takeover's row names the basket | **pass**: `REG01-000001`, receipt `Printed`; `sale.resume` on `parked_cart` under the sale's id, in a commit of its own; the sale's commit whole, 13 members, 13 delivery rows (03) |
| 7 | 6 planted defects | each turns the test that names its rule red | **pass**: 6 of 6 caught, each compiled. M01 and M02 put back exactly what shipped before (04) |

**One check needed a re-run.** `just pr` exited 1 when CI's `supply-chain` job got a 504 fetching its pinned Gitleaks scanner, which is a network failure unrelated to this change. The job was re-run. `validation` waits for this comment.

## Not validated here, and why

- **The basket discount and the department sale** have no shell command yet, so they have no end-to-end run. Their domain tests prove each names the basket, and M04 and M05 turn those tests red.
- **The receipt's time is printed in UTC**, as it is on every receipt today. That is an open question for the owner and outside this fix.
- **No screen** drives these commands yet. The harness calls the same plain functions the Tauri commands wrap.
