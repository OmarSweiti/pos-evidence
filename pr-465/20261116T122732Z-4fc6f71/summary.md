# 1.11.4, part two: the lock screen — validation at 4fc6f71

**Validated commit:** `4fc6f71`, this pull request's head, in a checkout with no change to any tracked file.

**Runtime:** the terminal's own `App` at this head, mounted in Chrome by an evidence-only Vite page on 127.0.0.1 and driven by the pinned `@playwright/cli` 0.1.22 (`--browser chrome`, no download).
- **Its till** is installed through Tauri's own `mockIPC`. It replays, as recorded, the answers a real till gave in part one's evidence run (#463): a register set up from the synthetic store-setup file, with its cashier's shift opened.
- **Only who is signed in moves,** checked against the synthetic fixture's codes and PINs.
- **It refuses as the real till would:** a machine with no register refuses `health_status` as `not_set_up`. A lock refused mid-payment is the register's `tender_collected` refusal.
- **Not driven:** the Tauri window itself. Its WebDriver is Windows and Linux only, and a real register here would use this Mac's Keychain, which evidence never touches.

**Data:** synthetic only: the fixture's store, staff and shift. Every PIN is shown as dots; the driver checks after each step that neither PIN's digits are anywhere in the page.

**Emulators:** none; no device is touched by this change.

## Commands

- `pnpm --filter terminal exec vitest run src/screens/Lock.test.tsx` (the step's `Done when`) and the numpad's tests, at this head; `pnpm -r test`, Biome, `tsc` and `just lint` at this head.
- `just pr`'s full gate at `e9f114a`, before the fix; CI's required checks at this head.
- Twelve planted defects, run at `e9f114a` and again at this head.
- The walk-through (`drive.sh`): every scene below, in Arabic, and English where marked.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | Tests and gate (board 01) | The `Done when` passes, with a different principal after the fast-switch fixture | Lock 8 and numpad 3 passed; every suite passed (money 10, ui 18, backoffice 3, terminal 74). The gate at `e9f114a`: 1139 workspace tests, builds, guards, secrets. CI at this head: on board 01. |
| 2 | Planted defects (board 02) | Each turns red the test aimed at it | 12 of 12 caught at this head. |
| 3 | The lock screen (01, 02) | The store, the register, the open shift's owner and date, what waits to sync; Arabic and English | As expected, with "waiting to sync: 6", the store's English name in English, and the keypad in the telephone layout in both. |
| 4 | A staff number, then a PIN (03) | The PIN as dots | 3001 shown; the PIN as six dots; its digits nowhere in the page. |
| 5 | A wrong PIN (04) | Refused without saying which half; PIN cleared, number kept | "رقم الموظف أو الرقم السري غير صحيح."; the till still signed out. |
| 6 | The right PIN (05) | Signed in, routed onward | The till holds 3001. Both routes show the Phase 0 smoke panel until their screens exist, with Lock and F12. |
| 7 | F12, then another person (06) | A fast user switch: signed out first, then the manager | The till signed out (the screen's sign-out ran under StrictMode twice, the second answered `not_signed_in` and treated as done), then holds 2001; the owner's number never sent. |
| 8 | No shift open (07) | "No shift open"; the waiting count | As expected; waiting to sync: 1. |
| 9 | A lock refused mid-payment (08) | Says so; only the way back | The refusal's words and "back to the till"; no field, no keypad; the till still holds 3001. |
| 10 | A machine not set up (09) | Says why; no keypad, no shift line | As expected; the sync count "–", since the till refuses `health_status`. |
| 11 | The 1920 profile at 125 % (10) | The same screen, centred | As expected. Measured: 12 numpad keys, the smallest 56 px tall; the numpad `ltr` in an `rtl` page; no `<input>` on the page. |

## Found here, and fixed at this head

The first run, at `e9f114a`, showed three things now fixed, each with an assertion that fails without the fix:
- **Arabic names in English lines** were not isolated. They are now `<bdi>`.
- **A refused lock** still showed an empty staff-number field.
- **A machine not set up** showed "no shift open".

## Not validated here, and why

- **The packaged Tauri window and WebKit:** only Chrome rendered this, for the reasons above.
- **Touch on a real panel:** keys were pressed by the driver.
- **The sale and shift-open screens** the routes lead to: 1.11.5 and 1.9.5's screen.
