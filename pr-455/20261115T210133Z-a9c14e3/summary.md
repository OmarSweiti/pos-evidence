# 1.8.5c, part one: the store-setup file — validation at a9c14e3

**Validated commit:** `a9c14e3`, this pull request's head, from a clean scratch worktree.
**Runtime:** `pos-db` (a library with no screen yet) and the till around it, driven by an evidence-only harness linked against this head's `pos-db` and `terminal` (`Till::open_in`, as `run` opens one). Each harness verb is one process.
**Data:** synthetic only. The setup file is `pos_db::fixtures::store_setup`'s: `SYNTH_` bands, 2001 dates, a source that says it is not an official catalogue, an invented business, store and staff. The OS credential store is stood in for by a file beside the scratch machine, so this Mac's Keychain is never touched. The recovery code goes to a paper file and is never printed. No PIN, PIN hash, key, code, id or local path appears in any image.
**Emulators:** none. This part writes no receipt and touches no device. The preview sale's receipt is not shown and was not inspected; receipts are 1.8.3's and 1.7's, and this part changes neither.

## Commands

- `cargo nextest run -p pos-db --lib store_setup::`, `--test audit the_travelling_payload`, `--test repository_transaction_api repository_transaction_api::`, each once in full and three times more.
- The harness: `custody`, `file <variant>`, `check`, `apply <pins>`, `show`, `till`, `preview <date>`.
- `verify-audit --database <copy of register.db>` (the repository's own binary; debug build, `POS_DB_KEY` from the stand-in store).
- `just pr`'s gate (`just pre-push`), quoted from its log at `a23e239`; GitHub's required jobs at `a9c14e3`.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | The tests (board 1) | 14 setup tests, the payload golden, 1.8.1's four transaction-guard tests pass, every time | All pass in all four runs. nextest flagged two setup tests LEAK in the third repeat (output still open just after the test exited, on a loaded machine); they passed, and different tests flagged on earlier runs. |
| 2 | The gate (board 1) | Lint, every workspace test, web builds, guards, secret scan green; CI's required jobs green | `just pre-push` passed in full at `a23e239` (1124 tests, 4 skipped). At `a9c14e3`, `rust`, `printer-conformance`, `guards`, `web`, `supply-chain`, `protected-paths` and `topology` succeeded; `validation` waits for this. |
| 3 | Custody (board 2) | A clean machine's first run completes custody and waits at `store_not_set_up`, every setup table empty | `new_machine`, then `store_not_set_up`; all 15 tables a setup writes hold 0 rows. |
| 4 | The prepared file (board 2) | `read` accepts it: store SYN-S1, register REG01, three people with their roles, three products; `Debug` shows codes and counts only | As expected. |
| 5 | Refusals at read (board 3) | Unapproved policy, pack changed after its pin, unregistered store, obligation pending evidence, missing GST evidence, missing tax number, price above ceiling and scale-label barcode refused by name; a PIN in the file, a phone as a number and a price as text refused by line and column; no refusal quotes the file | All ten refused as expected; "quotes" is false for every one, including the two values the JSON parser itself would quote. |
| 6 | PIN refusals (board 3) | A five-digit manager PIN and a missing cashier PIN refused by staff number, never naming the PIN; every table still empty | As expected; all 15 tables 0. |
| 7 | Set up (board 4) | The PINs checked and hashed before the transaction; everything written once; GST profile from the file; policy and pack approved; credential not enrolled; owner org-wide, staff at the store, each PIN opening its account; catalogue under its bands; one audit row by the owner with a complete envelope and no PIN, hash, tax number or name | As expected. |
| 8 | Once only (board 5) | The same file again refused; tables unchanged | Refused: the database already holds a business. |
| 9 | Audit chain (board 5) | `verify-audit` on a copy verifies the setup's row | VERIFIED (exit 0), one row, intact; no anchor supplied, as the tool says. |
| 10 | The till (board 5) | Without the device id file (part two's), the till still waits at `store_not_set_up` | As expected. |
| 11 | Preview of part two (board 5) | With the register row's device id written by hand: the till opens; the owner signs in and opens no sale; the cashier signs in, opens a shift and sells one product at the file's price | As expected. A first-start shift needs a typed business date (`business_date_required`, 1.9.4's clock rule); with it, receipt REG01-000001, 350 fils, cash. |
| 12 | Planted defects (board 6) | Every defect turns a test red | 29 of 29 caught at `a23e239`, whose library code this head keeps byte for byte. M17 re-run at `a9c14e3`: the strengthened test catches it on its own. |

## Not validated here, and why

- **A second full local gate at `a9c14e3`.** Started and stopped after an hour: macOS scanned each newly built test binary for many minutes before it could run. CI's required jobs at this head (the whole workspace suite among them) and the full local gate at `a23e239` stand instead; the only change between the two heads is inside the test module.
- **The 29 mutants at `a9c14e3`.** For the same reason, one mutant took 16 minutes there. They ran at `a23e239`; M17, the one the later commit is about, was re-run at this head.
- **The till's `store_setup` command, the device id file, and the plan's four `store_setup_` tests** are part two's; the preview writes the device id by hand only to show the written identity is enough to sell.
- **A real tax pack and an approved computation policy** do not exist yet (`ref/schema.md` §0003's two `⚠️ OPEN` items); everything here is the synthetic fixture.
