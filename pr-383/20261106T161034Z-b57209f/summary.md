# Validation: 1.4.11's second part, parking

**Commit validated:** `b57209f`, the pull request's head. It was checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **Three cart commands** in `apps/terminal/src-tauri/src/commands/cart.rs`, each proved through the command context:
  - `cart_park { label? }` under `sale.park`;
  - `cart_list_parked` under `sale.resume`;
  - `cart_resume { parked_id }` under `sale.resume`.
- **A resume onto a till that holds a basket sets that basket aside first, in the same transaction** (the owner's decision of 6 November 2026). It is parked if it has a line and dropped if not. A refused resume changes nothing.
- **Taking over another cashier's basket writes `sale.resume`** with its delivery envelope (#363). A cashier resuming their own basket writes nothing.
- **`ParkedCartRepository::set_aside`** is the one rule for setting a basket aside, used by sign-out, start-up and resume. It keeps the label the basket was last parked under.

No screen calls these commands yet; the sale screen is a later step. The runtime is shown through the shell's own code: a harness linked against this head's `terminal`, `pos-db` and `pos-domain`. Each step runs it as a new process. That process opens the register with `Register::open_with`, as `run` does through `Till`, and calls the plain function each command's Tauri wrapper calls.

**How it ran:**
- **The suites (01):**
  - the `Done when`, `cargo nextest run -p terminal --test ipc_contract --test cart`;
  - the parked-cart repository's tests;
  - every `terminal` and `pos-db` test;
  - clippy over both crates;
  - the frontier check;
  - then `just pr`'s gate at this head, which runs the test catalog.
- **The harness (02–05):**
  - Built `--offline` against the worktree, with a fresh synthetic register file per scenario.
  - Every step is its own process, and its exit status is recorded beside its output.
  - An action written `!action` must be refused. The board builder refuses to render any other outcome, and any audit row without its envelope.
  - A crash is a real `abort()`: no destructor runs and no one is signed out.
  - The database key is a synthetic test key passed to `open_with`, so the OS credential store is not touched.
- **The planted defects (06):**
  - 21 of them, each applied to the worktree.
  - Each was tested against the suite that names its rule, then restored with git.
  - No migration and no test was mutated.
- **One interruption.** The machine hibernated on a flat battery during the scenario run and woke on power about 14 hours later. Every step is its own process against its own file, so each scenario's steps ran as written. The boards show each one's output and exit status.

Data is synthetic throughout:
- an invented organisation and store, registered for GST, with prices including tax;
- register `REG01` with its shift open on 2026-11-05;
- two cashiers, "Synthetic Cashier" `C-100` and "Synthetic Second Cashier" `C-200`, and an owner, "Synthetic Owner" `O-100`;
- four `SYN-` products with opening stock.

A PIN is typed by the harness and printed nowhere. The board builder refuses to render a board holding a fixture PIN, the key or a local path.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal --test ipc_contract --test cart` | 23 pass | **pass**: 23 of 23 (7 contract, 16 cart), including `every_command_declares_the_arguments_its_handler_takes` and `no_command_argument_carries_a_price` (01) |
| 2 | The parked-cart repository, every `terminal` and `pos-db` test, clippy, the frontier | all pass | **pass**: 10 of 10, including `a_working_cart_is_set_aside_parked_or_dropped`; 74 of 74; 331 of 331; clippy clean; the frontier at 76 of 123 (01) |
| 3 | `just pr`'s gate at this head | lint, the test catalog, tests, web build, guards and the secret scan pass | **pass**: the catalog reconciles (96 cases); 923 passed, 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; no secret found (01) |
| 4 | Park with no cart, then with an empty one | each refused by name | **pass**: `cart_not_open`, then `nothing_to_park` (02) |
| 5 | Park under 41 Arabic letters, then under a label holding a line break | each refused, the basket left on the till | **pass**: `park_label_invalid` twice; the next park takes the same basket `…8dea` (02) |
| 6 | Park under `"  Blue bag  "` | parked under the trimmed label; the till free; listed with who parked it, its lines and its total; no audit row | **pass**: `label "Blue bag" · parked by Synthetic Cashier · 2 line(s) · total 2.850 JOD`; `the till holds no cart`; the file shows it `parked`, expiring with 2026-11-05; audit rows `(none)` (02) |
| 7 | In a new process, open an empty cart, then resume the basket | the empty cart dropped; the basket back with its lines at their captured prices | **pass**: the file holds only `…8dea`, `active`, 2 lines, total 2.850 as parked (02) |
| 8 | Park it again with no label | parked with no label | **pass**: `…8dea · (no label)`: a park sets the label it is given (02) |
| 9 | Build a one-line basket, then resume the parked one | the till's basket parked in the same step; the resumed one active; no audit row for a cashier's own basket | **pass**: the list then holds `…73eb`, 1 line, 1.250; `…8dea` active; the only audit row is the earlier sign-out's (02) |
| 10 | A cashier signs out holding a basket with an age-restricted line | the basket parked, the sign-out naming it | **pass**: `…e693 parked cashier C-100`; `auth.logout cause requested, parked_cart …e693 [enveloped]` (03) |
| 11 | A second cashier lists the basket and resumes it | listed under the first cashier's name; back as it was; the takeover audited | **pass**: `parked by Synthetic Cashier`; the age answer still `WAITING`; `active cashier C-200`; `sale.resume parked_by C-100, line_count 2 [enveloped]` (03) |
| 12 | Resume a basket no register parked | refused, nothing changed | **pass**: `parked_cart_unknown` (04) |
| 13 | Move a parked basket's business day to yesterday, then resume it | refused as expired, no longer listed, both baskets left where they were | **pass**: `parked_cart_expired`; `the parked list is empty`; the till still holds `…57d2`; `…7d9b` still `parked`; audit rows `(none)` (04) |
| 14 | An owner parks, lists and resumes | each refused for want of the grant | **pass**: `permission_denied` three times (04) |
| 15 | Resume a labelled basket, add a line, then the process dies | the next start-up sets it aside with its label and the new line; resumed, it is the basket left | **pass**: exit 134; `1 session(s) … closed, 1 basket(s) set aside`; `parked … 2 line(s) saved label "Blue bag"`; resumed at 1.600 (05) |
| 16 | 21 planted defects | each turns the test that names its rule red | **pass**: 21 of 21 caught, each compiled, and the worktree restored with no changed file (06). M17's replacement first failed to compile, so it tested nothing. It was rewritten as a value that is still always `None` and re-run alone, and then two sign-out tests went red. |

Every required check except `validation` is green at this head, `protected-paths` included: this branch touches no byte-pinned file.

## Not validated, and why

- **A screen driving these commands.** No screen calls them yet, so the app was not launched. The commands are shown through the plain functions their Tauri handlers wrap.
- **`parked_cart_claimed`**, a basket another till is working on. The harness drives one register. `a_refused_resume_leaves_both_baskets_where_they_were` marks the parked basket `active` under another session's nonce and asserts this refusal. Its closing assertions show both baskets left in place.
- **`training_mode_mismatch`.** No till opens a training cart yet, so the shell cannot meet one. The refusal is the domain's own, tested in `pos-domain`.
- **The price override and the department sale.** These are this step's third and fourth parts; the department sale waits on #330.
- **A park slip, and the end-of-day report of expired baskets.** No microstep's `Files:` line names a park slip yet, and nothing reads `ParkedCartRepository::expired` yet.
- **Opening with the key from the OS credential store.** The harness passes a synthetic key to `open_with`, so the credential store is never touched.
