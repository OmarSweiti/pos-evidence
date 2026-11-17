# Validation: 1.11.8 part one, `sale_reprint`

**Commit validated:** `8347db7`, the pull request's head, in the repository's checkout with no change in the tree.

**What changes at runtime:** a new command, `sale_reprint { sale_id }`, under `sale.reprint`, audited, with no approval. It writes a duplicate of a completed sale's receipt: the original's document plus the DUPLICATE watermark, citing the original, with its own queued print job and its audit fact behind one envelope, in one transaction. The print queue's `resolve_unknown` now writes its duplicate through the same two helpers. No screen calls the command yet; the post-sale panel is part two.

**How it ran:**
- **The suites (board 01):** run in the checkout at this head:
  - the reprint suite;
  - the print queue's own suite, `resolve_unknown`'s included, after the shared helpers;
  - the whole terminal crate;
  - clippy.

  `just pr`'s gate is quoted from its log at this same head.
- **The planted defects (board 02):** 7 on the committed tree at this head, each run against the reprint suite and reverted with `git checkout`. The tree was clean after every one.
- **The command on a real register file (board 03, and the two receipts 04–05):**
  - An evidence-only harness, linked against the checkout's own `terminal`, `pos-db`, `pos-domain` and `pos-hardware`, ran one process on a fresh register file built from the shell suites' synthetic shop.
  - Its printer is 1.7.12's emulated printer, bound through the register's own `printer.binding` setting and listening on 127.0.0.1. It records every document and drawer pulse it receives and renders each document as a PNG.
  - The print worker was driven one pass at a time.
- **The emulator profiles:** `emulator-80mm`, and `emulator-58mm` for the refusal of another printer.

**Data is synthetic throughout:** the fixture's store and register REG01, a bottle of oil at 1.250, and cashier C-100 under a synthetic name. The receipts carry no buyer.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 01 | Suites, clippy, gate | All green | Reprint 2 passed; print queue 19; terminal crate 218; clippy clean; the gate passed 1162 Rust tests. CI passed every required check at this head except `validation`, which waits for this evidence. |
| 02 | Planted defects | Each caught by the test aimed at it | 7 of 7. P2 is caught by the schema itself: an audit row no envelope carries is refused, so the reprint answers `storage_failed`. |
| 03 | A cash sale completes and prints | The original printed; the drawer pulsed once | `REG01-000001`, sent to the printer; one drawer pulse |
| 03 | `sale_reprint` | A duplicate citing the original, on its template and profile, queued; one `sale.reprint` audit fact naming it, behind its own envelope; the sale unchanged | All as expected. The worker printed it, and no drawer pulse followed. |
| 03 | `sale_reprint` again | A second duplicate citing the original, not the first duplicate; the same document | As expected: the two duplicates are byte for byte the same, and neither is the original |
| 03 | Refusals | Each by name, writing nothing | Unknown sale: `sale_unknown`. Store renamed since: `reprint_receipt_changed`. Signed out: `not_signed_in`. A register on the 58 mm profile: `reprint_printer_differs`. Artifacts and audit rows unchanged after each. |
| 04 | The original receipt, as the emulated printer rendered it | The sale's receipt | As expected |
| 05 | The duplicate, as the emulated printer rendered it | The same document with the copy mark, nothing else changed | "نسخة" under the title; every other line as the original's |

The second duplicate's image is byte for byte the first duplicate's (the same PNG digest), so it is not published again.

## Not validated here

- **A physical printer:** the emulated printer stands in for one, as `ref/hardware-and-receipts.md` §6b allows until milestone 5.8.
- **The post-sale panel that calls the command:** part two.
