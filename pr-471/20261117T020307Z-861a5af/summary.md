# Validation: 1.11.5 part one, the sale screen's commands

**Commit validated:** `861a5af`, the pull request's head, in the repository's checkout with no change in the tree.

**What changes at runtime:** five commands the sale screen calls, and the cart's generated TypeScript.
- **`catalog_search { query, limit }`** returns live, active products, at most 50, as `ProductView`.
- **`catalog_tiles`** returns this store's twelve most-sold products over thirty days, ranked by sales, then by SKU (the owner's decision of 16 November 2026).
- **`cart_set_qty { line_id, qty_milli }`** is screened for stock as an added line is; zero is not a removal.
- **`cart_void_line { line_id, reason, approval_id? }`:**
  - under `line.void`, audited, the reason a code from the store's list or its default (the owner's decision of the same day);
  - conditional on a merchant's policy, as an override is.
- **`cart_void_reasons`** gives the picker the store's list.
- **The domain's void payload now carries `amount_minor: 0`.** The schema holds a spent approval's amount to that field, so an escalated void could not be approved before.

There is no screen yet: the screen is part two. The runtime is shown through a harness linked against this head's `terminal`, `pos-db` and `pos-domain`.
- **Each step is a new process.** It opens the register with `Register::open_with` on the shell suites' own synthetic shop, as `run` opens it, with its clock at the time its command line gives.
- **It drives the shipped functions** that the Tauri commands wrap: `search`, `tiles`, `set_line_qty`, `void_cart_line`, `void_reasons`, the sign-in, the cart, payment and finalize, and `approve` (`auth_verify_pin`).
- **Nothing is stored by hand** except what a merchant or a store sets:
  - the store's void-reason list, as a setting row;
  - the hard block on negative stock;
  - a product made inactive.
- **No PIN, hash or id is printed.** Lines and approvals are named by the order they were made in.

**How it ran:**
- **The suites (01):** run in the checkout:
  - the new suites;
  - the IPC contract, approval and cart suites;
  - the generated-type drift tests and the domain's void tests;
  - every `terminal`, `pos-db` and `pos-domain` test;
  - clippy with `-D warnings`.

  Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–07):**
  - Built `--offline` against the checkout, with a fresh synthetic register file per scenario.
  - The board builder refuses a non-zero exit, an action whose outcome differs from the one written, a failed check, the synthetic PIN, an argon2 hash prefix, a UUID or a local path.
- **The planted defects (08):** 15, run on the committed tree at this head before `just pr`.
  - Each was run against its suite, then restored with git.
  - The tree was clean afterwards.

**Data is synthetic throughout:**
- the shell suites' shop, with register `REG01` and a shift open on 2026-11-05;
- oil (ten bottles at 1.250), cheese (half a kilogram at 8.500 a kilogram), tobacco (18+) and bread (no stock);
- cashier C-100, manager M-100 and the owner O-100, with one synthetic PIN.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 01 | Suites, clippy, gate | All green | `sale_commands` 6 passed. `product_most_sold` and `setting` 9. The contract, approval and cart suites 39. `terminal`, `pos-db` and `pos-domain` 1014. Clippy exit 0. The gate passed 1158 Rust tests, 4 skipped, with web, guards and secrets clean. |
| 02 | Search | By Arabic name and SKU, the age shown; operators are text; limit 0 finds nothing; an inactive product is not offered; nobody signed in is refused | Oil, cheese and tobacco (18+) found. `"(`, `OR` and limit 0 found nothing. Oil was not found once inactive. `not_signed_in` once signed out. |
| 03 | Tiles | None before a sale. After sales: oil, bread, cheese. The same thirty days on, none a day later | As expected, across three processes |
| 04 | Quantity | 3 then 1; 0 and 1.5 refused; under the hard block 11 refused and 10 kept | As expected (`qty_not_positive`, `qty_not_whole`, `stock_would_go_negative`) |
| 05 | Void | The default list; an off-list reason refused and nothing changed; voided once; the store's list replaces the default; an empty list refused; audit rows without names, behind envelopes | As expected. Two `line.void` rows, by C-100, with `amount_minor 0`; each check that the row names no product passed. |
| 06 | Escalated void | Refused without an approval. M-100 approves at the counter. Refused for another reason. Spent for its own, with the audit naming M-100 and approval #1 | As expected |
| 07 | The owner | No `line.void` | `permission_denied` |
| 08 | Planted defects | Each caught by the test aimed at it | 15 of 15 |

## Not validated here

- **The screen,** which is part two.
- **The Tauri IPC round trip.** The harness calls the plain functions the commands wrap, and `ipc_contract` holds each registered command's argument schema to its handler's signature.
- **No emulator is involved:** nothing here touches a device.
