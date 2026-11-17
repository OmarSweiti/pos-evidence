# Validation: 1.9.5 part one, Shift open

**Commit validated:** `5c2baf6`, the pull request's head, in the repository's checkout with no change in the tree.

**What changes at runtime:**
- **Shift open (ui-spec screen 2)** is where a person who signs in on a register with no open shift lands, in place of Phase 0's smoke panel.
  - The float is counted by denomination, in a grid of the register's own notes and coins, from the keyboard or the shared numpad.
  - Its total is the register's figure.
  - While the register trusts no clock, Open asks for the business day: the register's reading of the store's day in words, Previous day and Next day (never before the last shift's day), and Confirm.
  - Refusals are said in words. Once open, the sale screen follows.
- **The register:** a new inert read, `shift_open_prepare`, under `shift.open`. It answers the pieces, the counted float, the clock's day and the last shift's day, plus the generated TypeScript of the shift answers.

**How it ran:**
- **The suites (board 01):** run in the checkout at this head:
  - Shift open's suite, which holds this part's `Done when`;
  - the business-date helper and the whole terminal suite;
  - the register's `shift_open_prepare` test and the repository's last-business-day test;
  - the generated types held to the Rust;
  - the web build (`tsc`) and `just lint`.

  `just pr`'s gate is quoted from its log at `748c519`, with the one commit after it and the CI checks at this head.
- **The planted defects (board 02):** 25, each made on the committed tree and reverted with `git checkout`. The tree was clean after every one.
  - S1–S17 and B1–B2 ran with Vitest at this head.
  - R1–R6 ran with nextest at `853bce5`; no Rust file has changed since.
- **The walk (board 03, and screenshots 04–14):**
  - **The app:** the terminal's own `App` at this head, bootstrapped as its `main.tsx` does, without StrictMode as the production build runs. An evidence-only page mounts it, and the pinned `@playwright/cli` drives it in Chrome on 127.0.0.1.
  - **The till:** Tauri's own `mockIPC` answers with real registers' recorded answers. An evidence-only recorder, linked against this checkout's `terminal`, `pos-db`, `pos-domain` and `pos-hardware`, drove the register's own functions on fresh register files built from the shell suites' synthetic shop with no shift open, in the order the walk makes its calls. The page checks every call, nested arguments included, against the arguments recorded for it.
  - **The clock:** the shop's register clock reads 12:00 in Amman on 5 November 2026.
  - **The printer:** in the first scene, 1.7.12's emulated printer (`emulator-80mm`), bound through the register's own `printer.binding` setting and listening on 127.0.0.1. Its render of the first sale's receipt is image 15.
  - **The scanner:** the bottle was scanned with a burst sent through Chrome's DevTools protocol.
- **Profiles:** 1024 × 768 and 1536 × 864, in Arabic and English.

**Data is synthetic throughout:** the fixture's store and register REG01, a bottle of oil at 1.250, cashier C-100 and owner O-100 under synthetic names. The receipt carries no buyer. The PIN is never shown, and the log checks after every step that its digits are nowhere in the page.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 01 | Suites, build, lint | All green | Shift open 5 passed, the `Done when` among them; business date 3; terminal suite 119; `shift_lifecycle` 12 (terminal) and 11 (pos-db); types held; build and lint clean. The gate passed 1164 Rust tests at `748c519`; CI passed every required check at this head except `validation`, which waits for this evidence. |
| 02 | Planted defects | Each caught by the test aimed at it | 25 of 25 |
| 03 | The walk log and the recorder | Every call as recorded; nothing left on the tape | No call out of order or with other arguments in any scene; all three tapes played to the end; the PIN's digits never in the page |
| — | C-100 signs in on a register with no open shift | Shift open, not the sale | Shift open, with the register's eleven pieces at nought and a float of 0.000 |
| 04–07 | The float counted from the keyboard, key by key | Each key one `shift_open_prepare`; the float is the register's answer | Six preparations; 10.000 × 2, 5.000 × 4, 1.000 × 10, 0.500 × 6, 0.250 × 4 make 54.000, the register's figure. Arabic and English at both profiles. |
| 08–09 | Enter | `shift_open` with no date; the register asks for the day; the day step shows the clock's day in words | Refused `business_date_required`; "الخميس، 5 تشرين الثاني 2026" and "Thursday, 5 November 2026", with 2026-11-05. The recorder confirmed the refusal wrote nothing. |
| — | Next day, then Previous day, by touch | The day moves; the keyboard's focus stays put | 6 November, then 5 November; the focus stayed on the page |
| 10 | Enter | The shift opens on the 5th; the sale screen follows, naming the shift | `shift_open` with 2026-11-05 answered. The recorder wrote one shift and its `shift.open` row. The status strip names the cashier's shift. |
| 11 | The first sale: a scanned bottle of oil, Exact, Enter | It completes in the new shift | REG01-000001, printed with one drawer pulse. The recorder read the sale dated 2026-11-05 in its shift dated 2026-11-05. |
| 12 | A register whose last shift is dated the 6th, its clock reading the 5th | The day step starts at the 6th; Previous day disabled | As expected, with "Last shift: 2026-11-06"; the opening confirmed on the 6th. Off the tape, the recorder asked the register to open on the 5th: refused `business_date_before_last_shift`, writing nothing. |
| 13–14 | O-100, an owner, signs in on a register with no open shift | Refused in words; nothing to count | `shift_open_prepare` refused `permission_denied`; the refusal said; Open disabled; Lock offered |
| 15 | The first sale's receipt, as the emulated printer rendered it | The sale, dated the shift's day | REG01-000001, 2026-11-05, a bottle of oil at 1.250 |

**A wording fault the walk found, fixed before this head:** at `748c519` the header labelled whoever signed in as "Cashier", so an owner read "Cashier: Synthetic Owner". It now says "Signed in" (المستخدم). Screenshots 04–14 are from this head.

## Not validated here

- **The Tauri shell and a physical printer:** the walk runs the web build against real registers' recorded answers, and the emulated printer stands in for a device, as `ref/hardware-and-receipts.md` §6b allows until milestone 5.8.
- **A trusted clock:** the register trusts no clock yet, so every opening asks for the day. The trusted branch, which dates the shift itself and refuses a supplied day, is the register's, and its shell suite drives it (`a_trusted_clock_dates_the_shift_by_the_stores_own_day`).
- **Shift close:** part two.
