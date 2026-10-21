**Runtime change: yes.** How a receipt row sets its label against its value changes in `pos-hardware`'s layout. This validation renders the same receipt before and after, at both widths, and prints the 80 mm one through the real pipeline: 1.7.10's driver to `just printer-emulator`.

**Setup**
- **Code:** before is `development` at `25f008c`. After is the PR head, `292d3d4`, with a clean tree. Each is a scratch worktree, with a throwaway harness built against it.
- **80 mm, through the pipeline:** at each commit, `just printer-emulator epson-tm-t20iii-ethernet-80mm` ran in that worktree, and the harness printed the receipt to it through `NetworkPrinter`.
- **58 mm, from the renderer:** no seeded 58 mm printer profile can encode yet; each leaves its framing or width unknown.
- **The receipt:** the crate's own synthetic receipt model, with a fiscal block.
- **Screenshots:** headless Chrome with a throwaway profile, each rendered from the run's own files.

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | Print the receipt at 80 mm through the driver, before and after | Printed at both, and each emulator PNG equal to its render | ✅ `status() → Ready`, `print → Printed` at both (576 × 1238, then 576 × 1130). 0 dots differ between emulator and render at either commit (01) |
| 2 | The 80 mm fiscal row | before: the label broken; after: the label whole, the UUID on the row beneath, in the amount column | ✅ before: رقم الفاتورة الضريبية in five fragments, رقم / الفا / تورة / الضر / يبية, beside the UUID. After: the label whole on one row, the UUID beneath at the left margin (01) |
| 3 | What else changed at 80 mm | only the fiscal row | ✅ rows 0–965 identical, and the last 105 rows identical. The band that changed is the fiscal row, 168 rows before and 60 after (01) |
| 4 | The 58 mm date row | the label whole | ✅ before: التاريخ broken into ال / تار / يخ beside its timestamp. After: whole, with the timestamp beneath. **The issue didn't name this row; this run found it** (02) |
| 5 | What else changed at 58 mm | only the date row | ✅ rows 0–411 identical, and the last 757 rows identical. The fiscal row at 58 mm was already whole, because its UUID is wider than the paper (02) |
| 6 | The corrected receipt, whole | every label whole at both widths | ✅ both pages shown in full. Labels wrap only between words (58 mm's `GST16 16%` beside its rate triple). The 58 mm UUID, wider than the paper, continues on the next row as before (03) |
| 7 | The two named tests, at the head | pass | ✅ 2/2 (04) |
| 8 | `cargo nextest run -p pos-hardware` | pass | ✅ 108/108 (04) |
| 9 | `scripts/check-test-catalog.py` | reconciles | ✅ 96 cases, including the new case 96 (04) |
| 10 | Three old behaviours put back | each turns a named test red | ✅ Wrapping inside the narrow room fails both tests ("… with 6 px beside its amount", "رقم الفاتورة الضريبية is not whole at the start of any line at Mm80"). Dropping a label with no room fails the sweep at −24 px. With 58 mm checked first: "التاريخ is not whole at the start of any line at Mm58". The worktree was restored with no change (04) |
| 11 | `just pre-push` on the head | green | ✅ exit 0 in 113 s from a clean tree: 466 Rust tests (2 skipped), 79 web tests, lint, guards, build-web, and the full-history secrets scan (04) |

**Not validated here, and why**
- **58 mm through a printer.** It went through the renderer only, because no seeded 58 mm profile can encode yet. Every 58 mm page here is the renderer's.
- **A native reader's sign-off.** The corrected layout is in 03. Judging it is 1.7.5's golden review, which this fix unblocks rather than performs.
- **Any physical printer.** None is bought before milestone 5.8.
