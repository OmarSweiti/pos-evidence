# Validation: #442, 1.8.5b(c), the recovery sheet

Validated commit: `b96ce47` (the pull request's head), from a clean scratch worktree.

## How it was run

- **No screen exists for this part yet; the screen is part (d), after 1.11.3b.** The till was driven without one: an evidence-only harness, linked against this head's `terminal`, `pos-db` and `pos-hardware` and not part of the repository, starts the till with `Till::open_in`, as `run` does. It then calls what `provision_recovery_code`, `print_recovery_code` and `acknowledge_recovery_code` call, naming the printer in the JSON the screen sends. Each harness verb is one process.
- **Printer:** 1.7.12's emulated ESC/POS printer, under the emulator's 80 mm and 58 mm profiles (`emulator-80mm`, `emulator-58mm`), reached over TCP 9100 on 127.0.0.1. It runs in memory in the harness process, so a fault (out of paper) can be scripted per connection.
- **Machine:** a scratch directory per machine, started empty. A file beside it stands in for the OS credential store, so this Mac's real Keychain was never touched. Synthetic data only.
- **The code:** a shown code goes to a "paper" file and is never printed in the output; it appears masked (`#####-…`). A real code's sheet is never written to a file. The page the emulator printed is compared in memory, dot for dot, with the sheet of the code on the paper. The only sheet shown is a **specimen**: thirty zeros, key id `00000000`, printed with `RecoverySheet::print` directly. It belongs to no register.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-hardware recovery_sheet_` and `-p terminal recovery_code_provisioning_`, each once and three times more; `-p pos-db key_custody::`; every `pos-hardware` and `terminal` test; clippy; `just pr`'s gate | all pass | 7, 8 and 12 pass on every run; 289 pass; clippy exit 0; the gate is clean (1072 workspace tests). One LEAK, below |
| 2 | The specimen sheet printed to the emulator on 80 mm and 58 mm | one sheet each, cut; Arabic then English, the code large in three whole lines | printed, cut, 576x972 and 384x1036 dots; the pages are on board 02 |
| 3 | "New register", then the sheet with the printer out of paper | `recovery_sheet_not_printed` · out of paper; no paper; nothing sent again while nobody presses Print | as expected; 1 connection, 0 sheets, still 1 connection a second later |
| 4 | Print again with paper loaded | `()`; one sheet, cut; the page is the sheet of the code shown | `()`; 2 connections, 1 sheet, cut; dot for dot: true |
| 5 | Print once more | `recovery_sheet_already_printed`; printer not reached | as expected; still 2 connections |
| 6 | Files and rows after printing | unchanged files; no receipt artifact, print job, attempt or setting | unchanged: true; all four counts 0 |
| 7 | Another id; then confirm, then print | `provisioning_id_unknown`; confirmed; `recovery_code_not_shown` | as expected; the confirmed envelope opens with the paper's code |
| 8 | A second machine: printed, app stopped before confirming, started again | `code_not_confirmed`; a fresh code with its own sheet; the first code opens nothing | as expected; dot for dot: true; first code refused, fresh code opens |
| 9 | Printers the screen might name | emulator profiles usable; a real device's profile, a version not shipped and the simulator refused as `printer_unusable` with why; host name, version 0 and an unknown field never parse | as expected |
| 10 | 17 planted defects in a scratch worktree | each caught by its named test | 17 of 17 caught, every mutant compiled |

## Not validated, and why

- **No screen:** the provisioning screen is 1.8.5b's part (d), after 1.11.3b's design direction.
- **No physical printer:** every real device's profile is refused until milestone 5.8 qualifies one, so the emulator is the only printer a sheet can be printed through.
- **Memory wiping:** that the sheet's glyphs, dots and bytes are wiped is not observable from a test. It was reviewed by reading the code, and no mutant targets it.
- **One LEAK** on the first run of the `pos-hardware` filter (board 01): nextest marked `recovery_sheet_takes_the_code_as_the_screen_shows_it` leaky on the run right after its test binary was rebuilt. The test starts no thread and no process. It was not seen in the three runs after it, or in six more run by hand.
- **The Arabic wording** wants a native reader's review: board 02.
