# Validation: 1.11.7, the tender screen

**Commit validated:** `f2ec1a2`, the pull request's head, in the repository's checkout with no change in the tree.

**What changes at runtime:**
- **The tender screen (screen D4).** Cash is typed in fils or added up from the dinar notes; Exact takes what is asked; the collected list and what is left are always on screen; the cash rounding has its own line when it adjusts; the change is shown, and Complete finalizes the sale.
- **The sale screen** hands Pay's answer to the tender screen, and states a completed sale on the next basket.
- **The register:** `tender_current` (new), `PaymentSnapshot.quick_keys_minor`, and the generated TypeScript of the payment and finalize answers.

**How it ran:**
- **The suites (board 01):** run in the checkout at this head:
  - the tender screen's suite, which holds the step's `Done when`;
  - the sale screen's completed-sale test and the whole terminal suite;
  - the register's `tender_current` and quick-key test with the contract's;
  - the generated types held to the Rust;
  - the web build (`tsc`) and Biome.

  `just pr`'s gate is quoted from its log at `bf7419e`, with what came after it.
- **The planted defects (board 02):** 25 on the committed tree at this head: 21 in the screens (Vitest) and 4 in the register (nextest). Each was reverted with `git checkout`, and the tree was clean after every one.
- **The walk (board 03, and screenshots 04–18):**
  - The terminal's own `App` at this head, bootstrapped as its `main.tsx` does, on an evidence-only page served by Vite on 127.0.0.1. StrictMode is left off, as the production build runs.
  - Driven in Chrome by the pinned `@playwright/cli` at 1024 × 768, and at the 15.6-inch profile (1536 × 864): through the lock screen, the sale screen and the tender screen, in `App`'s own routing.
- **The till behind the page is the register's own answers, replayed:**
  - An evidence-only recorder linked the checkout's `terminal`, `pos-db` and `pos-domain`.
  - It drove the register's own functions on the shell suites' synthetic shop, in the order the walk calls them: `login_pin`, `open_cart`, `tiles`, `list_parked`, `resume_cart`, `begin_payment`, `add_cash`, `take_back`, `finalize`, `add_product`, `logout` and `current_payment`.
  - It wrote each answer or refusal as the webview receives it, and `state_of` and `health` for the two reads the screens poll.
  - The page replays the answers through Tauri's own `mockIPC`, checking each call against the arguments recorded for it.
  - Neither scene had a call that was not as recorded, and none of either tape was left unplayed.
  - The recording was made with the register at `bf7419e`. No Rust file has changed since: `bf7419e..f2ec1a2` touches only TypeScript and documents.
- **Hardware:** no device is touched. The fixture binds no printer, so Pay reads `no_printer`, and the screen says so before any money is taken. Each completed sale's receipt waits in the queue, and its drawer is to be opened by hand. Both are said on the next basket.

**Data is synthetic throughout:**
- the fixture's store and register REG01;
- synthetic products: a bottle of oil and 101 g of cheese, set aside earlier, then a bottle of oil;
- cashier C-100, under a synthetic name;
- one synthetic six-digit PIN, typed on the numpad and shown only as dots. The log records whether its digits appear anywhere in the page: they never do.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 01 | Suites, build, gate | All green | Tender 7 passed (the `Done when`: cash, overtender, split tender); the completed-sale test; terminal 106; register 21 with `the_tender_screen_reads_the_payment_and_its_notes`; the generated types held; both apps built; Biome clean. The gate passed 1160 Rust tests. CI passed every required check at this head except `validation`, which waits for this evidence. |
| 02 | Planted defects | Each caught by the test aimed at it | 25 of 25 |
| (walk) | C-100 signs in on the lock screen; F7 resumes the basket set aside | `auth_login_pin`, then the sale screen; the basket of oil and cheese brought back | As expected; TOTAL 2.108 |
| 04–05 | F4: payment begins (Arabic, English) | The amount due under the double rule; the printer's absence said in words; nothing collected; all of it left to pay | As expected: 2.108 due, "No printer is attached. The receipt will be kept and printed later." |
| 06 | 1, 0, 0, 0 typed, Enter | Digits enter as the figure's last (0.001 after the first); a part payment of 1.000; 1.108 left | As expected |
| 07 | The exact remainder typed, short of the coin step | Refused in words; nothing taken | `tender_underpaid`: "هذا المبلغ لا يغطي المتبقي بعد تقريب النقد." The typed amount is kept for correction. |
| 08 | Entry cleared; the dinar taken back | The list empty again; 2.108 left | `tender_remove`; as expected |
| 09 | Two one-dinar notes, Enter | The notes add to 2.000, taken as one tender; 0.108 left | As expected |
| 10 | Exact | What cash is asked (0.110) taken; settled; the rounding as its own line; Complete enabled | Rounding 0.002; change 0.000; Complete enabled |
| 11 | F4 | Finalized once; the next basket states the sale | Receipt REG01-000001, change 0.000 under the double rule, "the receipt waits for a printer, open the drawer by hand"; the tiles now rank the two products sold |
| 12–15 | A bottle of oil from the tiles, F4, a ten, Enter (Arabic, English, and the 15.6-inch profile in both) | Settled; change 8.750 under the double rule; no rounding line, since the rounding adjusts nothing | As expected |
| 16 | Enter | Finalized; the next basket states it | Receipt REG01-000002, change 8.750 |
| 17 | The till left mid-payment: the lock screen arrives | Sign-out refused, since cash is taken; the way back to the sale offered | `auth_logout` refused `tender_collected`, said in words, with "العودة إلى البيع" |
| 18 | Back to the sale | The sale screen finds the till paying, and the tender screen reads the payment it did not begin | `cart_open` refused `sale_being_paid`, then `tender_current`: 2.500 due, 0.500 collected, 2.000 left |

## What the evidence found, and what changed

**The register reports a rounding for every final cash tender, a zero adjustment included.** The first walk showed that a ten for 1.250 would have drawn "Cash rounding 0.000". A rounding that adjusts nothing now has no line. The overtender test, whose fake till now answers as the register does, holds it (T21).

The planted defects also found a gap: once settled, the pad was disabled but the keyboard's digits still typed. The screen now ignores them (T17).

## Not validated here

- **A physical printer, drawer and the Tauri window.** The screens ran in Chrome against recorded answers, not in the register's own window. The register's commands are exercised by its own suites (board 01).
- **Card tender** is Phase 2's (2.1); the post-sale panel is 1.11.8's.
- **`paper_out_warns_at_pay_not_after`** is 1.11.12's. Here the warning is shown for the fixture's real reading, `no_printer`.
