# Validation: 1.8.3's second part, the receipt model

**Commit validated:** `ec856ba`, the pull request's head. It was checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **`build_receipt_model(sale, ctx)`** (new, `crates/pos-domain/src/receipt.rs`) turns a `CompletedSale` into the `ReceiptModel` that 1.7.1 validates and 1.7.3 renders. It returns a `Result`.
- **`ReceiptContext`** carries what the sale cannot say: merchant, register code, cashier, language, fiscal standing, printed names, loyalty balance and footer.
- **`FiscalStanding`** decides the heading:
  - `Exempt` and `Cleared` print a sale;
  - `Uncleared` prints the non-fiscal acknowledgement.
- **Four new refusals:** `TenderUnlabelled`, `DiscountUnlabelled`, `LoyaltyWithoutCustomer` and `TrainingSaleCleared`. A `Money` error passes through.

No command calls the builder yet; 1.8.3's third part, the finalize, will. So the runtime is shown through a harness linked against this head's `pos-domain` and `pos-hardware`. Each scenario does four things:
1. It makes a completed sale through the domain's own transitions (`begin_tender`, `add_tender`, `begin_finalize`, `complete`), the calls the finalize makes.
2. It builds the receipt and prints every figure on it.
3. It checks that the receipt adds up, and runs `ReceiptModel::validate`.
4. It renders the receipt through 1.7.3's `render_receipt` into the dots a printer receives, shown beside each report.

**How it ran:**
- **The suites (01):** run in the worktree:
  - the receipt suite, `cargo nextest run -p pos-domain receipt::`;
  - the new property again at `PROPTEST_CASES=100000`;
  - every `pos-domain` test;
  - the renderer's `render::` tests in `pos-hardware` with every feature;
  - clippy over `pos-domain` with `-D warnings`;
  - the purity, acyclic-modules and property-name checkers, and the frontier.
- **The gate (01):** `just pr`'s gate is quoted from its log at `1c3d834`, the commit before the head. It covers clippy over the workspace, the test catalog, every test in the workspace, the web tests and build, every guard self-test and the history secret scan. The head, `ec856ba`, changes two test-fixture timestamps in `receipt.rs` and their comment, and nothing else. CI runs the whole gate again at the head.
- **The harness (02–07):**
  - Built `--offline` against the worktree. Each scenario is its own process, and its exit status is recorded beside its output.
  - A line written `!build` is a refusal the scenario expects.
  - The board builder refuses to render any of these:
    - an unexpected exit;
    - a receipt whose arithmetic check failed;
    - a built receipt the validator refused;
    - a buyer's name;
    - a local path.
- **The planted defects (08):**
  - 30 of them, each applied to `build_receipt_model` in the worktree.
  - Each was run against the builder's suite, then restored with git.
  - No test and no migration was mutated.
  - The property's seed files that the mutants left untracked were deleted before the suites ran, and the worktree was clean when they did.
- **Builds ran without debug information** (`CARGO_PROFILE_DEV_DEBUG=0`) to fit the free disk. That changes nothing a test observes.

**Data is synthetic throughout:**
- the merchant "مؤسسة اصطناعية للتجارة" and its store, with TIN `000000000`;
- register `REG01`, receipt `REG01-000042`, and the cashier "Synthetic Cashier";
- business day 2026-11-05, with the sale completing at 10:01:30 UTC;
- `SYN-` tax rows at 16% and 0%;
- a coin step of 5 fils, rounding to the nearest;
- a business buyer with TIN `000000001` and no name;
- a card whose masked number is `**** **** **** 0000` under the scheme `SYN`;
- a fiscal UUID of zeros.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-domain receipt::` | 30 pass: 1.7.1's 17 and the builder's 13 | **pass**: 30 of 30 (01) |
| 2 | `prop_a_receipt_adds_up_to_its_own_total` at 100,000 cases | passes, leaves no regression file | **pass**: 1 of 1, and 0 regression files (01) |
| 3 | Every `pos-domain` test, the renderer's tests, clippy and the four checkers | all pass | **pass**: 397 of 397; 41 of 41; clippy clean; each checker exits 0, and the frontier reconciles (01) |
| 4 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `1c3d834`: 957 passed with 4 skipped (the existing `#[ignore]` ones), 79 web tests, every guard self-test. The recipe's last step, `gh pr create`, was refused 401 because the gate outlasted the app's one-hour token. The PR was opened by hand at the same head (01) |
| 5 | Oil and 101 g of cheese, paid with 1.000 then 5.000 in cash. The receipt is in Arabic, at a store that owes an e-invoice and has not cleared the sale | lines at their gross; a +0.001 rounding line; total 2.110; change 3.890; the acknowledgement heading | **pass**: lines 1.250 and 0.859; subtotal 2.109; rounding +0.001; TOTAL 2.110; tenders 1.000 and 5.000; change 3.890; heading إشعار استلام غير ضريبي; all four checks PASS; `VALIDATE ok` (02) |
| 6 | The same sale in English, at a store exempt from e-invoicing | headed SALE, left to right, every amount the same | **pass**: heading SALE; `English LeftToRight`; the same figures; all four checks PASS (03) |
| 7 | Two teas with 10% off for a damaged pack, oil, and sugar overridden to 0.750, with 0.500 off the basket; a buyer's TIN captured | each discount beneath its line, under its reason's name; the override's price printed; the buyer block | **pass**: tea 4.000 with عبوة تالفة 0.400 and زبون دائم 0.321; oil 1.250 with 0.112; sugar `1 × 0.750` with 0.067; subtotal 6.000; discount 0.900; TOTAL 5.100; TIN 000000001, name (none); all four checks PASS (04) |
| 8 | Prices excluding tax; 2.000 in cash and the rest by card; the sale cleared; 58 mm paper | tax added to the total; the card charged the exact rest; no rounding and no change lines; SALE with its fiscal UUID | **pass**: subtotal 3.345, tax 0.160, TOTAL 3.505; card 1.505 with its masked number; no rounding or change line; the fiscal UUID beneath the tenders (05) |
| 9 | A training sale; then the same sale handed a cleared fiscal block | watermarked TRAINING; refused a fiscal block | **pass**: تدريب beneath the heading; `TrainingSaleCleared` (06) |
| 10 | A tender kind with no printed name, then a blank one; a loyalty balance with no customer; a discount reason with no name; a buyer with a blank TIN; then a customer attached | each refused by name; the balance prints for the customer | **pass**: `TenderUnlabelled { code: "cash" }` twice; `LoyaltyWithoutCustomer`; `DiscountUnlabelled { code: "damaged" }`; `BuyerWithoutTin`; then `build ok · LOYALTY 120 points` (07) |
| 11 | 30 planted defects | each turns the test that names its rule red | **pass**: 30 of 30 caught, each compiled, each by the test that names its rule (08) |

## Not validated here, and why

- **The register app was not launched.** No command or screen calls the builder until the finalize, 1.8.3's third part, so there is nothing to drive in the app. The harness calls the same public functions the finalize will.
- **No printer emulator was used.** Nothing here reaches a device. Each receipt image is `render_receipt`'s output, the raster the print queue sends to a printer.
- **No database is touched**, so there was no fresh-database step.

## Seen, and not changed here

- **The receipt prints its time in UTC.** It reads `2026-11-05T10:01:30.000Z`, which is 13:01 in Amman. That is 1.7.3's renderer (`layout.rs`, `issued_at.to_iso8601()`), unchanged by this pull request.
- **`print_queue::render_duplicate` cannot match a training sale's original.** It re-renders the model with no watermark and compares bytes. That defect is recorded in the 1.8.3 note for the finalize.
