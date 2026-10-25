**Validated at `31f9922`.** Microstep 1.2.4's shell half: the scan handler. 1.2.4 stays partially delivered; only the cart test remains.

`31f9922` is this branch brought up to date with `development` after #289. Since it was opened, the only change is #289's three `undici` lines in `pnpm-lock.yaml`, which no Rust code reads. Everything below was re-run at this head from a clean build.

**Setup**
- **Code under test:** a clean scratch worktree of this exact head.
- **Harness:** throwaway, outside the repository, linking this head's `terminal` crate (`terminal_lib::scan::handle_scan`), `pos-db` (with its `fixtures` feature) and `pos-domain`.
- **Database:** a fresh encrypted register database opened through `pos_db::open`, with the store built through the repositories:
  - a cola sold by the can and by the six-pack;
  - a white cheese at 8.500 JOD a kilogram under PLU 34567;
  - one deli scale with verified evidence, recorded with its envelope;
  - label layouts 22 (price), 23 (weight) and 24 (price, switched off).
- **Synthetic data only.**
- **How each scan was made:** handed to `handle_scan` as raw bytes, with the time, clock confidence and store passed in. That is the call the `catalog_by_barcode` command will make.
- **No app or emulator:** the IPC command is not wired yet, so there is no screen to drive.

| # | Step | Expected | Observed |
|---|---|---|---|
| 1–2 | The cola's can and six-pack codes | the product, ×1 and ×6 (E.78) | ✅ Board 01 |
| 3 | A price label on layout 22 | one package at the label's 0.416 JOD; about 49 g recorded as an estimated stock basis | ✅ Board 01 |
| 4 | A weight label on layout 23 | 1.250 kg at the catalogue price | ✅ Board 01 |
| 5 | The price label with one digit damaged | a checksum failure, rescan (E.40) | ✅ Board 01 |
| 6–7 | A barcode no product has; a label for a PLU no product has | unknown code: quick-add or department sale (E.39) | ✅ Board 01 |
| 8 | A label on the switched-off layout 24 | refused, naming the layout, not an unknown code | ✅ Board 01 |
| 9–10 | The price label on an untrusted clock; after the certificate expires | refused for the scale's reason | ✅ Board 01 |
| 11–13 | An empty scan, bytes that are not text, a 22-code too short to be a label | unreadable, rescan | ✅ Board 01 |
| 14–15 | After a maintenance event: the price label, then the cola | the label refused for the scale; the cola unaffected | ✅ Board 01 |
| 16 | The tests | the shell half's `Done when`, the shared layout choice, the database half's filter, and all of `pos-db`, `pos-domain` and the terminal | ✅ Board 02 |
| 17 | Eight mutants across the handler, `claiming_rule` and `rules()` | each caught | ✅ Board 03 |

CI at this head: `rust`, `printer-conformance`, `guards`, `web`, `supply-chain`, `protected-paths`, `topology`, `label`, and CodeQL with its five analyses are green, with **0 open code-scanning alerts**.

**Not validated here, and why**
- **The `catalog_by_barcode` IPC command and a screen.** The handler is a library function until the command is wired.
- **`price_embedded_line_is_one_unit_not_a_weight`.** It needs the cart's `AddLine::from_scan` (group 1.4).
- **A real scanner, scale or JSMO certificate.** The scans are raw bytes and the evidence references are synthetic.
