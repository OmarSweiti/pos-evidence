# Validation: 1.11.8 part two, the post-sale panel

**Commit validated:** `0a3f753`, the pull request's head, in the repository's checkout with no change in the tree.

**What changes at runtime:**
- **The post-sale panel (screen D5)** sits over the next basket and replaces 1.11.7's completed-sale notice. It shows:
  - the change, under the double rule;
  - the receipt number, and where the original went;
  - a drawer to open by hand, when there is no drawer;
  - Reprint, its one action, which calls `sale_reprint` and says the answer or the refusal.
- **It closes after about 3 seconds, or at once on any key or a touch outside it,** and takes nothing from the key. Tab reaches Reprint, and the keyboard's focus holds the panel's time. Escape always closes it.
- **The sale screen** draws the panel first, out of its flow.

No Rust file changed in this part.

**How it ran:**
- **The suites (board 01):** run in the checkout at this head:
  - the panel's suite, which holds the step's `Done when`;
  - the sale screen's completed-sale test and the whole terminal suite;
  - the web build (`tsc`), Biome and `just lint`.

  `just pr`'s gate is quoted from its log at `0b12814`, with the three commits after it and the CI checks at this head.
- **The planted defects (board 02):** 19 on the committed tree at this head, each run with Vitest and reverted with `git checkout`. The tree was clean after every one.
- **The walk (board 03, and screenshots 04–16):**
  - **The app:** the terminal's own `App` at this head, bootstrapped as its `main.tsx` does, without StrictMode as the production build runs. An evidence-only page mounts it, and the pinned `@playwright/cli` drives it in Chrome on 127.0.0.1.
  - **The till:** Tauri's own `mockIPC` answers with a real register's recorded answers. An evidence-only recorder, linked against this checkout's `terminal`, `pos-db`, `pos-domain` and `pos-hardware`, drove the register's own functions on fresh register files built from the shell suites' synthetic shop, in the order the walk makes its calls. The page checks every call against the arguments recorded for it.
  - **The printer:** in the first scene, 1.7.12's emulated printer (`emulator-80mm`), bound through the register's own `printer.binding` setting and listening on 127.0.0.1. The print worker was driven one pass at a time, and the emulator rendered each document as a PNG (images 17–18). The second scene has no printer.
  - **The scanner:** the scanner's burst was sent through Chrome's DevTools protocol, with each key stamped 5 ms after the last.
  - **The panel's timer:** the panel asks the window for one 3,000 ms timeout. For the screenshots, the page held that timeout so the walk's slower pace could capture the panel. For the second sale it left the timeout to the window, and timed the panel from its arrival in the page to its going.
- **Profiles:** 1024 × 768 and 1536 × 864, in Arabic and English.

**Data is synthetic throughout:** the fixture's store and register REG01, a bottle of oil at 1.250, and cashier C-100 under a synthetic name. The receipts carry no buyer. The PIN is never shown, and the log checks after every step that its digits are nowhere in the page.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 01 | Suites, build, lint | All green | Panel suite 5 passed, among them the `Done when` test: fake time closes the panel exactly once, at 3,000 ms, on the injected timer. Completed-sale test passed; terminal suite 111; build, Biome and lint clean. The gate passed 1162 Rust tests at `0b12814`; CI passed every required check at this head except `validation`, which waits for this evidence. |
| 02 | Planted defects | Each caught by the test aimed at it | 19 of 19 |
| 03 | The walk log and the recorder | Every call as recorded; nothing left on the tape | No call out of order or with other arguments; both tapes played to the end; the PIN's digits never in the page |
| 04–07 | A scanned bottle paid with a ten and completed, on the emulated printer | The panel over the empty next basket: change 8.750 under the double rule, receipt REG01-000001, "sent to the printer", no drawer line, one Reprint button | As expected, in Arabic and English at both profiles. The recorder saw the drawer pulse once and the original print. |
| — | Tab | Reprint is the first thing the keyboard reaches | Focus on Reprint; the panel stays |
| 08 | Enter on Reprint | `sale_reprint` for the sale; "a copy is waiting to print" | As expected. On the register: one `sale.reprint` audit row and one duplicate; the worker printed it (image 18), with no further drawer pulse. |
| 09 | Escape | The panel closes; the next basket is there | As expected; no command sent |
| 10 | The second sale, its panel left to the window's timer | It closes by itself after about 3 seconds | It lived 3,009 ms. Its timer was asked for once. |
| 11 | The third sale, then a scanner's burst while its panel shows | The burst closes the panel and lands in the new basket | 13 gaps of 5 ms; the panel closed; one `cart_add_scan` with the bottle's code; the oil in the basket at 1.250 |
| 12–13 | A scanned bottle, Exact, completed, with no printer | Change 0.000; "the receipt waits for a printer"; "open the drawer by hand" | As expected, in Arabic and English |
| 14–15 | Reprint clicked, the store renamed since the sale | The register's refusal, in words | `reprint_receipt_changed`, said as the store's details having changed, so the receipt cannot be reproduced exactly. The recorder confirmed it wrote nothing. |
| 16 | A touch outside the panel, on the search field | The panel closes, and the touch lands | The panel closed; the search field has the focus; no command sent |
| 17–18 | The original receipt and the duplicate, as the emulated printer rendered them | The same document, the duplicate marked as a copy | "نسخة" under the duplicate's title; every other line as the original's |

**A fault the walk found, fixed before this head:** at `0b12814` one panel asked the window for its timer twice. The screen beneath drew again as the next basket arrived and handed the panel a new function to close with, which restarted its time. The panel now keeps the newest function without restarting. `the_screen_drawing_again_does_not_restart_its_time` and planted defect P19 hold it. At this head, every panel's timer was asked for once.

## Not validated here

- **The Tauri shell and a physical printer:** the walk runs the web build against a real register's recorded answers, and the emulated printer stands in for a device, as `ref/hardware-and-receipts.md` §6b allows until milestone 5.8.
- **The panel's real-time close during the screenshots:** the panel's timeout was held for those images, as the walk log states. The unheld close is step 10.
