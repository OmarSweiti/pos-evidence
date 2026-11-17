# Validation: 1.11.5 part two, the sale screen

**Commit validated:** `d61efb0`, the pull request's head, in the repository's checkout with no change in the tree.

**What changes at runtime:** the register's sale screen (screen 3), drawn as the Ledger comp the owner chose. It has a status strip with an alarm band, the cart, search and the most-sold tiles, the totals band with TOTAL, and the action bar. It also covers void, park, resume and Pay, and a line that needs an age check. The register's `CartSnapshot` gains `tax_included`.

**How it ran:**
- **The suites (board 01):** run in the checkout at this head:
  - the screen's suite, which holds the step's `Done when`;
  - the whole terminal suite and `@pos/money`'s;
  - the register's tax test and its neighbours;
  - the generated types held to the Rust;
  - the web build (`tsc`) and Biome.

  `just pr`'s gate is quoted from its log at `0a79051`, with what came after it.
- **The planted defects (board 02):** 24 on the committed tree at this head: 22 in the screen (Vitest) and 2 in the register (nextest). Each was reverted with `git checkout`, and the tree was clean after every one.
- **The walk (board 03 and screenshots 04–20):**
  - The terminal's own `Sale` and `PaymentPending` at this head, routed as `App.tsx` routes them, on an evidence-only page served by Vite on 127.0.0.1.
  - Driven in Chrome by the pinned `@playwright/cli` at 1024 × 768, and at the 15.6-inch profile (1536 × 864).
- **The till behind the page is the register's own answers, replayed:**
  - An evidence-only recorder linked the checkout's `terminal`, `pos-db` and `pos-domain`.
  - It drove the register's own commands on the shell suites' synthetic shop, in the order the walk calls them: `open_cart`, `add_product`, `answer_age`, `set_line_qty`, `add_scan`, `void_cart_line`, `approve`, `park_cart`, `list_parked`, `resume_cart`, `begin_payment`, `back_to_cart`, `search`, `tiles` and `health`.
  - It wrote each answer or refusal as the webview receives it.
  - The page replays them through Tauri's own `mockIPC`, checking each call against the arguments recorded for it.
  - The walk log records any call that was not as recorded, and whether any of the tape was left unplayed. There were none, and none was left.
  - The recording was made with the register at `0a79051`. No Rust file has changed since: `0a79051..d61efb0` touches only TypeScript, CSS and documents.
- **Hardware:** no device is touched. The scanner is a burst of keys, sent through Chrome DevTools with each key stamped 5 ms after the last. The page records the gaps it measured: 5 ms each.

**Data is synthetic throughout:**
- the fixture's store and register REG01;
- products named for the fixture, with no real brand;
- cashier C-100 and manager M-100, under synthetic names;
- one synthetic six-digit PIN, typed on the numpad and shown only as dots. The log records whether its digits appear anywhere in the page: they never do.

The strip's two alarms, printer unavailable and receipts waiting, are the register's own `health_status` for the fixture: the fixture binds no printer, and the four baskets sold to fill the tiles queued their receipts.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 01 | Suites, build, gate | All green | Screen 13 passed (the `Done when` among them); terminal 98; `@pos/money` 12; register 42 with the tax test; the generated types held; both apps built; Biome clean. The gate passed 1159 Rust tests. At this head, CI passed every required check except `validation`, which waits for this evidence. |
| 02 | Planted defects | Each caught by the test aimed at it | 24 of 24 |
| 04 | The screen arrives (Arabic) | The till's basket, the most-sold tiles, the register's totals; the strip with store, register, what waits to sync, whose shift, time, Lock; alarms in a band, in words | As expected. Lines 2.500 and 0.350, subtotal 2.850, "منها ضريبة" 0.393, TOTAL 2.850 د.أ under the double rule. The till was asked `cart_open` and `cart_list_parked`, once each. |
| 05 | Tobacco from a search: blocked; then F4 | The line is in with its age question; Pay disabled and says why; F4 asks the register nothing | As expected. F4 showed "أكّد عمر المشتري أولًا." and the till received nothing. |
| 06 | Age confirmed | Pay enabled | `cart_confirm_age` with `confirmed true`; Pay enabled |
| 07 | The oil line selected, + | Quantity 3, line 3.750, TOTAL 6.600 | As expected (`cart_set_qty` with 3000) |
| 08 | A full code no product carries, Enter | Refused in words; nothing changes | "الرمز غير معروف." (`scan_unknown`); the basket unchanged |
| 09 | A scanner's burst while the search field has focus | A line added; the field left as it was | The oil line added and selected; the field empty; the burst measured at 5 ms gaps |
| 10 | Bread selected, Del | The line menu names the line and lists the store's reasons in words | As expected; then "صنف خاطئ" voided it (`reason wrong_item`) |
| 11 | Tobacco's menu, "damaged", under a policy naming `line.void` | The register asks for a manager, and the approval modal shows what is approved | `approval_required`; the modal shows "حذف سطر من السلة", the line, the reason and who asks |
| 12 | M-100 and the PIN on the numpad | The PIN as dots only | `••••••`; no PIN digits in the page |
| 13 | Approve | `auth_verify_pin`, then the void at once with that approval | Exactly that. The approval the void sent is the one `auth_verify_pin` answered; TOTAL 5.000. |
| 14 | F6 park, F7 resume | Parked with a notice and an empty basket; the list says who parked it and its total | "عُلّقت السلة."; the list shows 2 أصناف, Synthetic Cashier, 5.000 د.أ |
| 15 | The basket brought back | The parked basket as it was | Byte for byte the same image as 13 |
| 16 | Pay | Payment begins on the register; the holding screen | `tender_begin`; the holding screen. "Back to the basket" then ran `tender_back_to_cart`, and the screen arrived again with the basket. |
| 17–18 | The 15.6-inch profile, Arabic and English | The same screen, laid out for the wider window | As expected |
| 19–20 | A store pricing tax on top, nothing sold yet, Arabic and English | Lines add up to the subtotal; the row says "الضريبة" / "Tax"; TOTAL holds it; the tiles say where they will come from | 2.500 + 0.350 = 2.850, Tax 0.456, TOTAL 3.306; the empty-tiles text in both languages |

Steps 04 to 16 are in Arabic at 1024 × 768. English at 1024 × 768 is shown by 20. Five more shots were taken and inspected but are not published, because of the 20-image cap: 04 in English, the search's results, bread voided, parked before the list, and back to the basket. The walk log on board 03 records each of them.

## What the evidence found, and what changed

The first walks, at `0a79051` and `47ca82a`, found four faults. Each is fixed in this pull request:
- **The screen asked the register again whenever the app redrew it** (a language switch, for one). It now arrives once. The render test holds this (S21).
- **A tax-on-top line showed its total with the tax inside,** so the column did not add up to the subtotal. It now shows its gross. The render test holds this (S22).
- **At 1024 pixels, two alarms pushed Lock and the clock off the strip.** Alarms now have their own band beneath it.
- **The search field drew a second focus ring** around its bare input. There is now one ring, around the field.

Two more problems came from the testing itself, not the screen:
- **CI found a race in a test:** it clicked a tile when only the tiles' caption had been drawn. The test now waits for the tile.
- **In one walk, the driver's typed scan lost its first digit.** Keys typed by Playwright are stamped only as the page handles each one, and the first two arrived over 30 ms apart. The burst is now sent with each key carrying its own time, as a scanner's keys do.

## Not validated here

- **A physical scanner and the Tauri window:** the screen ran in Chrome against recorded answers, not in the register's own window. The register's commands are exercised by its own suites (board 01).
- **The tender screen** is 1.11.7's. Pay stops at the holding screen.
- **`keymap.test.ts`'s two Enter tests** failed together once in seven local runs of the terminal suite during this work. They passed in every other run, and in every run of the file alone. They use real time, and this pull request does not touch the file.
