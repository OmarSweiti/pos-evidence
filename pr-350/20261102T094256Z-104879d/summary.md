# Validation: harfrust 0.15.0, with shaping ported to its API

**Commit validated:** `104879d`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- **No screen to drive.** Receipts are rendered by `pos-hardware` before any printer sees them, so the evidence is the rendered pages themselves. The emulated network printer is exercised by CI's printer-conformance command, run here too.
- **One throwaway harness source, built twice,** offline in release against each tree's own `Cargo.lock`: once against `development` at `dccf2a3` (harfrust 0.13.3), once against this head (0.15.0). Each build renders, through the public `render_receipt`, the same synthetic models. It writes each page's raw dots, which are then compared bit by bit.
- **Planted defects:** each one changed one place in `crates/pos-hardware/src/render/layout.rs` in the worktree, ran every `pos-hardware` test with all features, and was restored. The run ends with "restored: 0 changed files", and the 0.15 harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-hardware render::layout::tests::` (1.7.3's shaping and layout tests) | all pass, the new vocabulary test and the pinned Arabic word among them | **pass**: 25/25 (01) |
| 2 | CI's `printer-conformance` command | passes, where #345 failed | **pass** (01) |
| 3 | CI's `rust` job, first step: clippy over the workspace, `-D warnings` | clean, where #345 failed | **pass** (01) |
| 4 | `pos-hardware` with all features; the workspace | no failure | **pass**: 112/112 and 790/790, with the existing `#[ignore]` tests skipped (01) |
| 5 | The synthetic receipt in Arabic and in English, on 80 mm and 58 mm paper, rendered with 0.13 and with 0.15 | byte-identical pages | **pass**: 4 of 4, 0 differing dots (02) |
| 6 | A receipt of 22 synthetic product names (joining, lam-alef, diacritics, the Arabic comma, digits, units, Latin brands, an accented letter), on both papers, in Arabic and in English | byte-identical pages | **pass**: 3 of 3, 0 differing dots (02) |
| 7 | Two of those pages, side by side | identical to the eye too | **pass** (03) |
| 8 | Ten planted defects in the ported code | nine caught; the tenth, an unreachable refusal, survives | **as expected** (04): nine caught, worktree restored |

## Notes

- **The tenth defect survives by design.** It replaces "refuse the page" with "drop the run" in the one place the shaper can refuse. But every run the layout builds is a fresh buffer with its direction set and no plan, so no input reaches that refusal, and no test can tell the two apart. It's listed so the survivor is on the record. The refusal is kept because the never-panic rule rules out `unwrap`, and a dropped line would be silent.
- **The new vocabulary test catches six of the nine.** The defects it catches range from a script never guessed (letters left unjoined) to a lost mark offset. It's what will notice the next shaper that moves a glyph, until 1.7.5's goldens exist.
- **The product-name receipt keeps the fixture's totals,** so its sums don't add up. It exists to put many words through the shaper and the wrap, and `render_receipt` validates a document's structure, not its arithmetic.

## Not validated, and why

- **A physical printer:** none is bought before the hardware qualification gate (`ref/hardware-and-receipts.md` §6b). The emulated printer is covered by step 2.
- **1.7.5's golden receipts:** not delivered yet.
