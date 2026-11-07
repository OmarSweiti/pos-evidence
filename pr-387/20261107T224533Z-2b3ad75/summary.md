# Validation: 1.8.3's first part, paying for a basket

**Commit validated:** `2b3ad75`, the pull request's head. It was checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **Five commands** in `apps/terminal/src-tauri/src/commands/tender.rs`, each proved through the command context under `sale.create`:
  - `tender_begin` screens the basket's stock once more, prices it under the store's own pack and policy, fixes the coin step, and journals the checkout before any money is taken;
  - `tender_add_cash { amount_minor }` takes cash, settles at the store's coin step and gives change, each tender journalled before it is answered;
  - `tender_remove { tender_id }` takes a tender back, audited as `tender.remove`, with its delivery envelope;
  - `tender_back_to_cart` returns to the basket while nothing is taken, and abandons the journal;
  - `sale_in_flight_state` names every sale journalled at the register.
- **The till holds a sale being paid for apart from a basket.** The cart commands that would change it, open another, park it or resume one are refused `sale_being_paid`.
- **The owner's two decisions of 7 November 2026:**
  - a sign-out with nothing taken parks the basket, and one asked for with a tender taken is refused;
  - a forced sign-out, or a restart, leaves the sale journalled for 1.8.4's recovery half.
- **`TenderTypeRepository::is_active`** (new, `pos-db`) decides which payment kinds the register takes.

No screen calls these commands yet. The runtime is shown through the shell's own code: a harness linked against this head's `terminal`, `pos-db` and `pos-domain`. Each step runs it as a new process. That process opens the register with `Register::open_with`, as `run` does through `Till`, start-up recovery included, and calls the plain function each command's Tauri wrapper calls.

**How it ran:**
- **The suites (01):** run in the worktree:
  - this part's suites, `cargo nextest run -p terminal --test tender --test cart --test ipc_contract`;
  - the new `pos-db` tender-kind suite;
  - every `terminal` test;
  - the frontier check.

  Then `just pr`'s gate at this same head: clippy over the workspace with `-D warnings`, every test in the workspace (`pos-db`'s 337 included), the web tests and build, the guards, the secret scan and the test catalog. The full `pos-db` suite and clippy were not run a second time in the worktree. The gate had run both at this head, and building them again into the kit's directory would not have fitted the disk.
- **The harness (02–05):**
  - Built `--offline` against the worktree, with a fresh synthetic register file per scenario.
  - Every step is its own process, and its exit status is recorded beside its output.
  - Every start-up prints what recovery found.
  - An action written `!action` must be refused. The board builder refuses to render any other outcome, and any audit row without its envelope.
  - Where a step changes the register's file by hand, it says so: the hard block, cash switched off, an account deactivated, a journal damaged.
  - The database key is a synthetic test key passed to `open_with`, so the OS credential store is not touched.
- **The planted defects (06):**
  - 28 of them, each applied to the worktree.
  - Each was tested against the suite that names its rule, then restored with git.
  - No migration and no test was mutated.
- **Builds ran without debug information** (`CARGO_PROFILE_DEV_DEBUG=0`) to fit the free disk. That changes nothing a test observes.

Data is synthetic throughout:
- an invented organisation and store, registered for GST, with prices including tax;
- its approved policy rounding cash to 5 fils, to the nearest;
- register `REG01` with its shift open on 2026-11-05;
- two cashiers, "Synthetic Cashier" `C-100` and "Synthetic Second Cashier" `C-200`, and "Synthetic Owner" `O-100`;
- three `SYN-` products:
  - oil at 1.250, ten in stock;
  - cheese at 8.500 a kilogram, half a kilogram in stock;
  - bread at 0.350, none in stock.

A PIN is typed by the harness and printed nowhere. The board builder refuses to render a board holding a fixture PIN, the key or a local path.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal --test tender --test cart --test ipc_contract` | 38 pass | **pass**: 38 of 38 (10 tender, 21 cart, 7 contract), including `no_command_argument_carries_a_price` and `every_command_declares_the_arguments_its_handler_takes` (01) |
| 2 | The tender-kind suite, every `terminal` test, the frontier | all pass | **pass**: 2 of 2; 89 of 89; the frontier at 76 of 123 (01) |
| 3 | `just pr`'s gate at this head | clippy, the test catalog, every test, web build, guards and the secret scan pass | **pass**: clippy clean; the catalog reconciles (96 cases); 944 passed, 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; no secret found (01) |
| 4 | Oil and 101 g of cheese, then payment begins | journalled; cash asked at the coin step | **pass**: total 2.108, cash asked 2.110; the journal holds the sale at version 0 (02) |
| 5 | While it is paid for: add a line, open a cart, begin again | each refused; the screen still shows the basket | **pass**: `sale_being_paid` three times; `current` shows the basket at 2.108 (02) |
| 6 | Cash 1.000, then exactly the 1.108 still owed | the part payment taken exactly; exact cash short of the coin step refused | **pass**: owed 1.108, cash asked 1.110; `tender_underpaid` (02) |
| 7 | Cash 5.000, then 0.005 more | settled once at the coin step, with change; nothing more taken | **pass**: `ROUNDING: owed 1.108, cash asked 1.110, adjustment +0.002`; change 3.890; settled; `sale_settled`; the journal at version 2 with both tenders (02) |
| 8 | Take back the first tender, then the second | each audited with its envelope; the rest taken again | **pass**: the 5.000 alone settles 2.108 at 2.110, change 2.890; two `tender.remove` rows, `[enveloped]`, naming 0.000 and then 2.108 owed (02) |
| 9 | Back to the basket, add a line, sign out | the journal gone; the basket parked | **pass**: total 3.358; journal `(none)`; the basket `parked`; `auth.logout` names it and no sale (02) |
| 10 | Begin with no basket, then an empty one; tender with payment not begun | each refused by name | **pass**: `cart_not_open`, `nothing_to_tender`, `payment_not_begun` three times (03) |
| 11 | Bread, of which there is none, goes to payment; then the store blocks a short basket | flagged while allowed; refused once blocked; the basket stays | **pass**: the stock flag on the payment answer; then `stock_would_go_negative`; `current` still shows the bread (03) |
| 12 | Cash of nothing, of −0.005, more than can be counted, an unknown tender, back with money taken, cash at a register that takes none | each refused; one tender journalled | **pass**: `tender_not_positive` twice, `tender_amount_invalid`, `tender_unknown`, `tender_collected`, `tender_type_disabled`; the journal at version 1, one tender of 1.000 (03) |
| 13 | The owner tries every command | refused for want of `sale.create` | **pass**: `permission_denied` five times (03) |
| 14 | Sign out while paying with nothing taken | the basket parked, the journal abandoned | **pass**: journal `(none)`; the basket `parked`; `sale_in_flight none` (04) |
| 15 | Resume, pay 0.500, sign out | refused; nothing changes | **pass**: `tender_collected`; the journal at version 1; the basket `active`; no new sign-out row; `sale_in_flight_state` shows it `ON THIS TILL` (04) |
| 16 | Take the tender back, sign out | the basket parked again | **pass**: one `tender.remove` row `[enveloped]`; journal `(none)`; the basket `parked` (04) |
| 17 | A cashier with 0.500 taken is deactivated; the next command | signed out; the sale left journalled, not parked | **pass**: `user_inactive`; the journal holds the sale; the basket `active`; `auth.logout cause user_inactive · parked_cart none · sale_in_flight …3d49 [enveloped]` (05) |
| 18 | A second cashier lists sales in flight, begins their own, and the process dies | the first named on no till; their own on this till | **pass**: the next start-up found it, `1 sale(s) left mid-payment`; `on no till`, then `ON THIS TILL` (05) |
| 19 | Restart | the dead session closed; both sales left; the till empty | **pass**: `1 session(s) closed, 0 basket(s) set aside, 2 sale(s) left mid-payment`; `the till holds no cart`; both baskets `active` (05) |
| 20 | Damage both journalled snapshots, then restart | both still left for recovery; the list says it cannot read them | **pass**: `2 sale(s) left mid-payment`; `checkout_unreadable`; the journal shows both as unreadable; both baskets `active` (05) |
| 21 | 28 planted defects | each turns the test that names its rule red | **pass**: 28 of 28 caught, each compiled, and the worktree restored with no changed file (06) |

Every required check except `validation` is green at this head, `protected-paths` included: this branch touches no byte-pinned file.

## Not validated, and why

- **A screen driving these commands.** No screen calls them yet, so the app was not launched. The commands are shown through the plain functions their Tauri handlers wrap.
- **Finalizing a paid sale.** `sale_finalize` is 1.8.3's third part. A settled sale here is journalled and waits; nothing completes it yet.
- **Resuming a sale left mid-payment.** That is 1.8.4's recovery half, after 1.8.3. Steps 17 to 20 show such sales are kept and named, not what becomes of them.
- **Any tender but cash.** Card, CliQ and the rest are Phase 2's, and `0005` seeds them inactive.
- **A real stock shortfall arising during payment.** No command in this part moves stock. The refusal at `tender_begin` is shown by switching the store to its hard block over a basket already short; the stock screen it runs is the same one.
- **Opening with the key from the OS credential store.** The harness passes a synthetic key to `open_with`, so the credential store is never touched.
