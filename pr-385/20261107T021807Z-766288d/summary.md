# Validation: 1.4.11's third part, the price override

**Commit validated:** `766288d`, the pull request's head. It was checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **`cart_override_price { line_id, unit_price_minor, reason, approval_id? }`** in `apps/terminal/src-tauri/src/commands/cart.rs`. It is proved through the command context under `price.override`, which the grid gives a shift lead and a manager, never a cashier.
  - The domain's `override_price` holds the price, given the product's `StockPosition::cost_basis()` as its cost.
  - Every override, and every return to the captured price, writes a `price.override` audit row in the transaction that saves the cart.
- **Reasons come from the store's list.** `SettingRepository::price_override_reasons` reads `price.override_reasons` at the store, or the default `displayed_price`, `damaged`, `price_match`, `catalogue_error` (merchant decision 3.7). `displayed_price` is always allowed. A list that cannot be read is `setting_invalid`, never the default.
- **No second person.** `ipc::context::merchant_policy` escalates nothing (the owner's decision of 6 November 2026). The registry row is `Conditional`, so a merchant policy that escalates the override asks for an approval bound to the line and the price.
- **An audit row spends the approval its intent names.** The shell's audit writer puts the `approval_consumption` row in the audit row's own envelope, in the same transaction.
- **`CartSnapshot`'s lines carry `captured_price_minor` and `override_reason`** beside the price they charge.

No screen calls this command yet. The runtime is shown through the shell's own code: a harness linked against this head's `terminal`, `pos-db` and `pos-domain`. Each step runs it as a new process. That process opens the register with `Register::open_with`, as `run` does through `Till`, and calls `override_line_price`, the plain function `cart_override_price`'s Tauri wrapper calls.

**How it ran:**
- **The suites (01):**
  - the `Done when`, `cargo nextest run -p terminal --test ipc_contract --test cart`;
  - the new `pos-db` setting suite;
  - every `terminal` and `pos-db` test;
  - clippy over both crates with `-D warnings`;
  - the frontier check;
  - then `just pr`'s gate at this head, which runs the test catalog.
- **The harness (02–05):**
  - Built `--offline` against the worktree, with a fresh synthetic register file per scenario.
  - Every step is its own process, and its exit status is recorded beside its output.
  - An action written `!action` must be refused. The board builder refuses to render any other outcome. It also refuses any audit row or approval consumption without its envelope, and any consumption outside its audit row's envelope.
  - The database key is a synthetic test key passed to `open_with`, so the OS credential store is not touched.
- **The escalated path (05).** The Tauri wrapper always passes the register's own policy, which escalates nothing. So the harness's `escalate` step hands `override_line_price` a policy that escalates `price.override`, as the cart suite does. Its `approve` step issues a manager's approval with the domain's `ApprovalHandle::issue`, under the manager's own `price.override` grant. It stores the approval behind its envelope with `ApprovalRepository::insert`. No manager PIN is typed; the approval screen is 1.11.4b.
- **The planted defects (06):**
  - 24 of them, each applied to the worktree.
  - Each was tested against the suite that names its rule, then restored with git.
  - No migration and no test was mutated.
- **Builds ran without debug information** (`CARGO_PROFILE_DEV_DEBUG=0`) to fit the free disk. That changes nothing a test observes.
- **Two incidents in the kit, neither in this pull request's code:**
  - **The first scenario run hung in the harness.** Its `table` step took the register's database lock twice, a `Mutex` that is not re-entrant. The harness was fixed to use the connection it already held. It was rebuilt from an empty build directory against the same worktree, and all four scenarios were re-run from fresh files. The boards show that second run. The suites on board 01 are from the first run, at the same head; the harness is not part of them.
  - **`just pr` could not open the pull request after its gate.** The gate passed and pushed this head, but it took 68 minutes. The engineering app's token, minted before the gate, had expired, so `gh pr create` was refused with HTTP 401. The recipe's remaining steps were run by hand with a fresh token: open the pull request, file its card, watch its checks. The gate's result on board 01 is that run's own.

Data is synthetic throughout:
- an invented organisation and store, registered for GST, with prices including tax;
- register `REG01` with its shift open on 2026-11-05;
- "Synthetic Cashier" `C-100`, "Synthetic Shift Lead" `L-100` and "Synthetic Manager" `M-100`;
- three `SYN-` products:
  - oil at 1.250 with a cost of 0.100;
  - cheese sold by weight from a scale's price label;
  - bread with no recorded cost.

A PIN is typed by the harness and printed nowhere. The board builder refuses to render a board holding a fixture PIN, the key or a local path.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal --test ipc_contract --test cart` | 28 pass | **pass**: 28 of 28 (7 contract, 21 cart), including `conditional_privilege_cannot_cross_threshold_without_approval`, `every_command_declares_the_arguments_its_handler_takes`, `no_command_argument_carries_a_price` and the five new override tests (01) |
| 2 | The setting suite, every `terminal` and `pos-db` test, clippy, the frontier | all pass | **pass**: 4 of 4; 79 of 79; 335 of 335; clippy clean; the frontier at 76 of 123 (01) |
| 3 | `just pr`'s gate at this head | lint, the test catalog, tests, web build, guards and the secret scan pass | **pass**: the catalog reconciles (96 cases); 932 passed, 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; no secret found (01). nextest marked one test leaky, the existing argon2 known-answer test `pos-db auth::tests::the_profile_hashes_to_its_known_answer`. It passed, this pull request does not touch it, and it was not leaky in the three gates before. |
| 4 | A shift lead overrides two bottles of oil from 1.250 to 1.000, reason `damaged` | charged 1.000 each, the captured 1.250 kept beside it; audited with the cost floor | **pass**: `× 1.000 = 2.000 OVERRIDE damaged, captured at 1.250`; audit `reason damaged · 1.000 (was 1.250, captured 1.250) · floor 0.100 · by L-100 · approver none [enveloped]` (02) |
| 5 | The shelf showed 0.050, below the oil's cost: `displayed_price` at 0.050 | charged; the floor does not apply | **pass**: `× 0.050 = 0.100 OVERRIDE displayed_price`; audit `floor none · displayed yes [enveloped]` (02) |
| 6 | Back to 1.250, reason `price_match` | the override cleared, and the clearing audited | **pass**: the line at 1.250 with no override; audit `reason price_match · 1.250 (was 0.050) [enveloped]`; the file holds the line `captured at 1.250` and nothing more (02) |
| 7 | With a ceiling of 1.500 on the oil, try each refusal: a line not in the cart; free text; a code not on the list; 1.501 for `price_match` and for `displayed_price`; 0.099; −0.001; `displayed_price` at the current 1.250; the uncosted bread lowered; the scale-labelled cheese | each refused by its own code | **pass**: `line_unknown`, `override_reason_unknown` twice, `price_above_ceiling` twice, `price_below_floor`, `price_negative`, `displayed_price_not_lower`, `price_no_recorded_cost`, `price_label_not_overridable` (03) |
| 8 | Store `"not json"` as the store's list, then override | refused, never read as the default | **pass**: `setting_invalid: the store's list of override reasons cannot be read` (03) |
| 9 | After every refusal | the cart as it was, no audit row | **pass**: the three lines at 1.250, 0.350 and 0.416; `price.override` audit rows `(none)` (03) |
| 10 | A cashier overrides | refused for want of `price.override` | **pass**: `permission_denied`; still no audit row (03) |
| 11 | The store's list set to `["loyalty_2026"]` | `damaged` refused; the store's code accepted; the uncosted bread raised; `displayed_price` still accepted | **pass**: `override_reason_unknown`; `OVERRIDE loyalty_2026` on the oil at 1.000 (floor 0.100) and on the bread at 0.400 (`floor none`, a raise); `OVERRIDE displayed_price` at 0.050 (04) |
| 12 | A manager overrides under the same list | accepted, audited under the manager | **pass**: `reason loyalty_2026 · 1.100 · floor 0.100 · by M-100 [enveloped]` (04) |
| 13 | With a policy that escalates, override with no approval | refused | **pass**: `approval_required` (05) |
| 14 | Offer a manager's approval for 0.900 to an override at 1.000 | refused before anything is written | **pass**: `approval_mismatch: the approval is for another amount` (05) |
| 15 | Offer an approval for 1.000 given for `price_match` to an override for `damaged` | refused as it is spent; the cart, the audit row and the consumption all rolled back | **pass**: `approval_mismatch: the approval was given for another reason`; the line still at 1.250; audit rows `(none)`; both approvals `unspent` (05) |
| 16 | Offer an approval bound to this line, 1.000 and `damaged` | accepted; the audit row names the manager and the approval; the approval spent on the line, in the audit row's envelope | **pass**: `× 1.000 OVERRIDE damaged`; audit `by L-100 · approver M-100 · approval …9ff4 [enveloped]`; `approval …9ff4: SPENT on line …2a54, its consumption in its audit row's own envelope`; the other two still `unspent` (05) |
| 17 | Offer the same approval again | refused as spent | **pass**: `approval_used` (05) |
| 18 | 24 planted defects | each turns the test that names its rule red | **pass**: 24 of 24 caught, each compiled, and the worktree restored with no changed file (06) |

Every required check except `validation` is green at this head, `protected-paths` included: this branch touches no byte-pinned file.

## Not validated, and why

- **A screen driving this command.** No screen calls it yet, so the app was not launched. The command is shown through the plain function its Tauri handler wraps.
- **`department_price_not_overridable` and `amount_discount_fixes_price` through the shell.** No command at this head can make the line either one guards:
  - a department line waits on the department sale, 1.4.11's fourth part;
  - an amount off waits on a command that takes one.

  The domain's `an_override_is_refused_by_name` holds both refusals. The shell's mapping of them to their codes is reached by no test yet.
- **An approval issued through the approval PIN screen** (1.11.4b), and an expired approval. The harness issues approvals valid for 120 seconds and spends them within that window.
- **A stored merchant escalation policy** (decision 1.8). The register's own policy escalates nothing, and the escalated path is driven with a policy the harness hands in.
- **A label reprint for a displayed-price override.** The queue arrives with migration `0014` at 4.6.2. Until then the audit row with reason `displayed_price` is the record.
- **Reason lists at the organisation or register scope.** Only the store's row is read, and no precedence among the three scopes is decided.
- **Opening with the key from the OS credential store.** The harness passes a synthetic key to `open_with`, so the credential store is never touched.
