# Validation: 1.4.11's first part, building a basket

**Commit validated:** `feacb8f`, the pull request's head. It was checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **Five cart commands** in `apps/terminal/src-tauri/src/commands/cart.rs`: `cart_open`, `cart_current`, `cart_add_line`, `cart_add_scan` and `cart_confirm_age`.
  - Each proves `sale.create` through the command context.
  - Each answers with `CartSnapshot`, the cart priced by `price_cart` under the store's own settings, in minor units.
  - None takes a price.
- **The registry refuses a money argument** unless a `PRICE_ARGUMENTS` line allows it on that command, and the row keeps that line's control.
- **The till's working cart is on disk from its first moment** (the owner's decision of 5 November 2026):
  - It is an `active` `parked_cart` row, and every change is saved before it is answered.
  - A sign-out, requested or forced, sets a basket with lines aside in the parked list and drops an empty one. Start-up does the same with what a crash left.
- **`pos-db`** gains `TaxPolicyRepository::pricing_settings`, `ParkedCartRepository::hold` and `discard`.

No screen calls these commands yet; the sale screen is a later step. The runtime is shown here through the shell's own code: a harness linked against this head's `terminal`, `pos-db` and `pos-domain`. Each step runs it as a new process. That process opens the register with `Register::open_with`, as `run` does through `Till`, and calls the plain function each command's Tauri wrapper calls.

**How it ran:**
- **The suites (01):**
  - the `Done when`, `cargo nextest run -p terminal --test ipc_contract --test cart`;
  - the database half's tests;
  - every `terminal` and `pos-db` test;
  - clippy over both crates;
  - the test catalog and the frontier check;
  - then `just pr`'s gate at this head.
- **The harness (02–04):**
  - Built `--offline` from the worktree's `Cargo.lock`, against a fresh synthetic register file per scenario.
  - Every step is its own process, and its exit status is recorded beside its output.
  - An action written `!action` must be refused. The board builder refuses to render any outcome other than the one written, and any audit row without its envelope.
  - A crash is a real `abort()`: no destructor runs and no one is signed out.
  - The database key is a synthetic test key passed to `open_with`, so the OS credential store is not touched.
- **The planted defects (05):**
  - 21 of them, each applied to the worktree.
  - Each was tested against the suite that names its rule, then restored with git.
  - No migration and no test was mutated.

Data is synthetic throughout:
- an invented organisation and store, registered for GST, with prices including tax;
- register `REG01` with its shift open on 2026-11-05;
- two cashiers, "Synthetic Cashier" `C-100` and "Synthetic Second Cashier" `C-200`;
- four `SYN-` products with opening stock;
- a deli scale whose evidence reference is marked synthetic.

A PIN is typed by the harness and printed nowhere. The board builder refuses to render a board holding a fixture PIN, the key or a local path.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal --test ipc_contract --test cart` | 17 pass | **pass**: 17 of 17, including `no_command_argument_carries_a_price` and `every_command_declares_the_arguments_its_handler_takes` (01) |
| 2 | The database half, every `terminal` and `pos-db` test, clippy, the catalog, the frontier | all pass | **pass**: 18 of 18; 68 of 68; 330 of 330; clippy clean; both checks reconcile, the frontier at 76 of 123 (01) |
| 3 | `just pr`'s gate at this head | lint, tests, web build, guards and the secret scan pass | **pass**: 916 passed, 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; no secret found (01) |
| 4 | A cashier opens a cart, then opens again | an empty cart on the open shift; the same cart again | **pass**: the same sale both times, total 0.000 JOD (02) |
| 5 | A bottle by product, three more as a quantity, the bottle's code, the six-pack's outer code | catalogue prices; the outer code adds six; tax inside every total | **pass**: lines of 1, 3, 1 and 6 at 1.250; total 13.750 with 1.895 GST inside (02) |
| 6 | Eleven bottles against ten on hand, the store allowing and flagging | sold and flagged | **pass**: `STOCK FLAG SYN-OIL-1L: on hand 10.000, the basket takes 11.000` (02) |
| 7 | An unknown product, 1.5 and 0 of a counted product, an unknown code, a damaged label, an empty scan | each refused by name, the cart unchanged | **pass**: `product_unknown`, `qty_not_whole`, `qty_not_positive`, `scan_unknown`, `scan_checksum_failed`, `scan_unreadable` (02) |
| 8 | A deli label under the register's own clock; the same label handed a trusted clock | refused with the scale's reason; then one package at the label's price | **pass**: `scan_scale_refused`: "the register's clock is not trusted…"; then `SYN-CHEESE 1.000 package × 0.416`, unflagged against half a kilogram (02) |
| 9 | The file, then a sign-out | the saved basket is the one shown; a sign-out parks it and names it | **pass**: `active … 5 line(s) saved`, then `parked`; `auth.logout cause requested, parked_cart …70da [enveloped]` (02) |
| 10 | A tobacco pack; yes; yes again; a second pack; no | waits, opens, refused, waits, removed; each answer audited | **pass**: `WAITING`, `confirmed`, `age_not_pending`, then the second pack removed; `age.confirm` and `age.decline` rows, both enveloped (03) |
| 11 | Bread with none on hand, allow-and-flag | sold and flagged, shown again by `cart_current` | **pass**: `STOCK FLAG SYN-BREAD: on hand 0.000, the basket takes 1.000`, twice (03) |
| 12 | Under a hard block: ten bottles of ten; an eleventh; a bread | the first accepted; the others refused, the cart unchanged | **pass**: 10 bottles at 12.500; `stock_would_go_negative` twice; `cart_current` still one line (03) |
| 13 | A process dies holding a basket of two lines | the next start-up closes the session and parks the basket; the till opens empty | **pass**: exit 134; `1 session(s) … closed, 1 basket(s) set aside`; `the till holds no cart` (04) |
| 14 | A process dies holding an empty cart | the next start-up drops it | **pass**: exit 134; `1 empty cart(s) dropped`; only the earlier basket remains (04) |
| 15 | A second cashier's basket, then a sign-out | parked, the audit row naming it | **pass**: `…927d parked cashier C-200`; `auth.logout cause requested, parked_cart …927d` (04) |
| 16 | A cashier deactivated with a basket on the till | the next command signs them out and parks the basket | **pass**: `user_inactive`, then `not_signed_in`; `…df35 parked`; `auth.logout cause user_inactive, parked_cart …df35 [enveloped]` (04) |
| 17 | 21 planted defects | each turns the test that names its rule red | **pass**: 21 of 21 caught; every mutant compiled, and the worktree was restored with no changed file (05) |

`protected-paths` is red by design on this pull request: `scripts/check-test-catalog.py` is byte-pinned, and this branch retires one `PLANNED` name in it. Every other required check is green at this head.

## Not validated, and why

- **A screen driving these commands.** No screen calls them yet, so the app was not launched. The commands are shown through the plain functions their Tauri handlers wrap.
- **A deli label priced on a real register.** No clock is trusted until 1.1.9's database half persists trusted time. `Register::clock_confidence` answers `Untrusted`, so every label is refused with the scale's reason (step 8). The trusted case is the same function handed a trusted clock, which no register holds yet.
- **Resuming a parked basket.** `cart_resume` and `cart_list_parked` are this step's second part. Here a parked basket is shown in the register's file.
- **The price override and the department sale.** These are this step's third and fourth parts; the department sale waits on #330.
- **Opening with the key from the OS credential store.** The harness passes a synthetic key to `open_with`, so the credential store is never touched.
