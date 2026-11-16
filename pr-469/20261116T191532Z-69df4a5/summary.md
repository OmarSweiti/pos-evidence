# Validation: 1.11.4b part two, the manager approval modal

**Commit validated:** `69df4a5`, the pull request's head, in the repository's checkout with no change in the tree.

**What changes at runtime:** the approval modal (screen 7) and `MoneyDisplay`, the first money on a screen.
- **The modal shows exactly what is approved:** the action, the thing, the amount in money, the reason and who is asking.
- **The approver gives a staff number, then a PIN.** The person asking is refused as their own approver before any PIN is asked for.
- **It runs the bound command at once, once, with the approval's id** (the owner's decision of 16 November 2026). A lapsed or changed approval is asked for again.
- **Money shows at the currency's exponent with its label,** د.أ in Arabic and JOD in English (the owner's decision of the same day).
- **The lock screen's entry pieces are shared** and its behaviour is unchanged.

No screen hosts the modal yet: the sale screen (1.11.5) and the stock and PIN-reset screens will. So it is shown through an evidence-only page in Chrome:
- **The terminal's own `ApprovalModal` at this head,** mounted over a plain page.
- **Its till is installed through Tauri's own `mockIPC`.** It answers with what a real till answered: every approval, refusal and command result was recorded from the register's own `approve`, `reset_pin`, `adjust` and `override_line_price` by part one's harness, on the shell suites' synthetic shop.
- **The fake keeps the register's rules.** It checks the shop's synthetic staff numbers and its one synthetic PIN, counts wrong PINs as 1.6.2's policy does, and binds each approval to its amount.
- **It numbers each approval it issues,** so that one is told from another; nothing else is changed from what was recorded.

**How it ran:**
- **The suites (01):** run in the checkout:
  - the modal's suite, which holds the step's `Done when`;
  - `MoneyDisplay`'s suite;
  - the whole terminal suite and `packages/ui`'s;
  - the web build (`tsc`) and Biome.

  `just pr`'s gate is quoted from its log at `3c84994`, the head it opened the pull request at, with what followed (below).
- **The planted defects (02):** 15 of them, made on the committed tree at this head.
  - Each was run against its test file and reverted with `git checkout`.
  - The tree was clean after every one.
- **The walk (03, and the screens 04–16):**
  - Chrome was driven by the pinned `@playwright/cli` 0.1.22, at 1024 × 768 and at the 15.6-inch profile (a 1536 × 864 viewport).
  - The page is served by Vite on 127.0.0.1 only.
  - The log records each step's alert, the two fields, whether either synthetic PIN's digits are anywhere in the page, and every command the till received. A PIN appears only as its length, and an approval's id only as which of those issued it is.

**What happened after the gate, said plainly:**
- **At `3c84994`, CI's `web` job passed all 88 tests, then failed** on an unhandled `ReferenceError: window is not defined`, reported against `catalog.test.ts`. React work had been left pending as a test file ended, and ran after jsdom was gone. This Mac never reproduced it.
  - **The fix:** a step in the test harness lets that work run before each test ends, and the modal's last test now waits for the modal to finish.
- **At `554fc71`, the web build refused** the harness's bare `setImmediate`. It is now read from the global with its type stated.
- **At `69df4a5`,** CI's `rust`, `web`, `guards`, `supply-chain`, `printer-conformance`, `protected-paths`, `topology` and `label` all passed. That is one green run, which cannot prove a timing race is gone, only that it did not recur.

**Data is synthetic throughout:**
- the fixture shop's line of oil at 1.250, overridden to 1.000;
- shift lead L-100, managers M-100 and M-200, and cashier C-100, under synthetic names;
- one synthetic six-digit PIN.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 01 | Suites, build, gate | All green | Modal 11 passed (the `Done when` among them), `MoneyDisplay` 3, terminal 88, `packages/ui` 18. Both apps built and Biome was clean. The gate passed 1149 Rust tests, one flagged leaky (`auth_errors_carry_no_pin_and_no_hash`, not touched here). |
| 02 | Planted defects | Each caught by the test aimed at it | 15 of 15 |
| 04–05 | The override as asked, Arabic and English | Action, thing, 1.000 د.أ / 1.000 JOD, reason, asker by name and role; two columns mirrored | As expected; values in ink, labels muted |
| 06 | The asker types their own staff number | Refused before any PIN; nothing sent | Self-refusal shown; the till received no call |
| 07 | A cashier, with the right PIN | The register refuses them; another approver is asked for | `permission_denied` words; staff field cleared |
| 08 | The manager's PIN | Dots only | `••••••`; no PIN digits in the page |
| 09 | A wrong PIN | Refused; staff number kept, PIN asked again | As expected |
| (walk) | The right PIN | `auth_verify_pin`, then the bound command once with approval #1 | Exactly that; the modal closed with "done" |
| 10–11 | The operation becomes 0.900 while 1.000 is approved | The command refuses; the modal asks again; approved at 0.900, spent | `approval_mismatch` words; then approval #2 bound at 900 and spent |
| 12 | The till finds the first approval lapsed | Asked again; the second approval is spent | As expected; #1 and #2 each sent once |
| 13 | Four wrong PINs, then the right one | The register makes the next attempt wait | The wait's words; no approval issued |
| 14–15 | A PIN reset (Arabic) and a stock adjustment (English) | No amount shown, exact zero bound | No amount row; amount 0 sent; each command run once |
| 16 | The 15.6-inch profile | Fits; keys ≥ 48 px | 12 keys, the smallest 56 px; no button under 48 px; no input; the dialog inside the viewport |

## Not validated here

- **A screen that hosts the modal,** which arrives with 1.11.5 and the stock and PIN-reset screens.
- **The Tauri IPC round trip itself.** The page calls the commands through Tauri's `invoke` against `mockIPC`; part one's evidence ran the register's side.
- **The receipt's printed currency label,** which waits for the receipt layout to name the currency.
