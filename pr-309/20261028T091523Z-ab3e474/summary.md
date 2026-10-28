# Validation: microstep 1.4.3, adding, changing and voiding a cart line

**Commit validated:** `ab3e474`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- `AddLine`, `add_line`, `set_qty`, `void_line` and `check_age_confirmed` are pure functions: no database, device, emulator or screen is involved.
- A throwaway harness, not in the repository, called the pull request's own functions and printed what they returned. It ran as a release build, for the sweep. Every id, name, code and amount is synthetic: a product's Arabic name is `صنف اصطناعي` ("synthetic item") and its number.
- Each planted defect changed one statement of `crates/pos-domain/src/cart.rs` in the worktree and was restored, proptest seed files included. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain cart::tests::` and `--test cart_ui` (the `Done when` this PR gives 1.4.3, since the plan had none) | 16/16 and 1/1 | **pass** (01) |
| 2 | The domain crate, then the workspace | no failure | **pass**: 271/271; 666/666 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | Two teas by product, then the catalogue reprices and renames tea, then the line's quantity changes | the line keeps 1.250 and its captured name | **pass** (02) |
| 4 | A department amount swapped into a tea line; a line with no product; a price in dollars on a dinar cart | each refused by name | **pass** (02) |
| 5 | A product deactivated with stock remaining (E.38), by product and by barcode, and the line captured before | adding refused both ways; the captured line still changes, at its price | **pass** (02) |
| 6 | An age-restricted pack beside an ordinary line (E.69) | the pack is in the cart, blocked, and `check_age_confirmed` refuses naming it | **pass** (02) |
| 7 | Quantities 0, −1, 1.5 of a counted item, 1.5 kg of cheese | the first three refused by name, the last accepted | **pass** (02) |
| 8 | A six-pack's outer barcode, a weight label, and a price label printed for 0.416 (1.2.4's last test) | six units; 1.250 kg at the catalogue price; one `Package` at 0.416 with 49 g as the stock basis | **pass** (03) |
| 9 | That label's quantity set to 2; a code nothing claims, a damaged label, two products on one code | each refused by name | **pass** (03) |
| 10 | `void_line` on one of two lines | the line removed; an intent with `line.void`, the reason code, the actor, the moment, and the line's ids, quantity and price, no name; a second void refused | **pass** (04) |
| 11 | Sweep: every quantity from −3.000 to 30.000 in thousandths, for each unit, by product, by pack quantity and by `set_qty` | zero or less refused, a fraction of a counted unit refused, everything else accepted, no fractional counted line in any accepted cart | **pass**: 495 015 attempts, 0 wrong outcomes (05) |
| 12 | Sixteen planted defects | each caught | **pass** (06): all sixteen caught, worktree restored |

## Notes

- A first run of the planted defects, at the same commit, ended "restored: 1 changed files". The source was restored each time, but the run script left proptest's untracked seed directory behind. The script now removes it after each defect, and this evidence is from the re-run.

## Not validated, and why

- **Confirming an age** (1.4.3b), **tendering** (1.4.8), and **the command that calls these from the screen**, with its `line.void` permission check (1.4.11).
- **A department line** (1.4.12): `add_line` and `set_qty` refuse one, by design.
