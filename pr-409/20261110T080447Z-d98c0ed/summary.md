# Validation: 1.10.5's stock commands, an adjustment asked for, approved and posted at the till

**Commit validated:** `d98c0ed`, the pull request's head. It was checked out clean in a scratch worktree with its own build directories.

**What changes at runtime:** three new commands.
- **`stock_on_hand { product_id }`, under `sale.create`:** the store's on-hand for a product, and the cost a unit is carried at. A cost never recorded is shown as unknown, never as zero.
- **`stock_adjust_prepare { product_id, qty_delta_milli, reason_code, note? }`, under `stock.adjust`:** stores the request under the id its event will carry, with the digest of exactly what was asked, who asked, and when. Nothing moves yet.
- **`stock_adjust { stock_event_id, approval_id }`, under `stock.adjust`, audited, and privileged on every call:**
  - The approval names that event, exactly zero, and the request's digest, which is recomputed from the stored row before the effect.
  - Only the person who asked posts the request.
  - The event is dated by the register's open shift.
  - The event, its `stock.adjust` audit row and the approval's consumption travel in one envelope, and the request is removed.

The runtime is shown through a harness linked against this head's `terminal`, `pos-db` and `pos-domain`.
- **Each step is a new process.** It opens the register with `Register::open_with`, as `run` opens it, on the shell suites' own shop file built by `pos_db::open`.
- **It drives the shipped commands:** sign-in, the cart, tender and finalize, and `on_hand`, `prepare_adjustment` and `adjust`, the plain functions the Tauri commands wrap.
- **No shipped command issues an approval yet:** `auth_verify_pin` is 1.11.4b's. So `approve` stores one the way that screen will: issued from the approver's own proof of `stock.adjust`, against the request as it is stored, behind its envelope.
- **No PIN, hash or id is printed.** Requests and approvals are named by the order they were stored in.

**How it ran:**
- **The suites (01):** run in the worktree:
  - 1.10.5's `Done when`, except its screen: `cargo nextest run -p pos-db opening_stock_`, `cargo nextest run -p pos-db --test stock_adjustment` and `cargo nextest run -p terminal commands::stock::tests::`;
  - the IPC contract;
  - every `terminal` and `pos-db` test;
  - clippy over both crates with `-D warnings`.

  Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–04):**
  - Built `--offline` against the worktree, with a fresh synthetic register file per scenario.
  - The board builder refuses any of the following: a non-zero exit, an action whose outcome differs from the one written, a failed check, the synthetic PIN, the word `argon2`, a UUID, or a local path.
- **The planted defects (05):** 17 of them, run at this head before `just pr`.
  - Each was run against the suite its rule belongs to, then restored with git.
  - The worktree was clean afterwards.
- **Builds ran without debug information** (`CARGO_PROFILE_DEV_DEBUG=0`).

**Data is synthetic throughout:**
- **The shop:** the reference chain's fixture merchant and its store, with register `REG01` and a shift open on it dated 2026-11-05 (none in scenario C).
- **The stock:** ten bottles of oil and half a kilogram of cheese posted as opening stock at 0.100 a unit, and no bread.
- **The people:** cashier C-100, managers M-100 and M-200, and the owner O-100, with one synthetic PIN.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | 1.10.5's `Done when`, except its screen | the opening-stock tests, the database suite and the commands' suite pass | **pass**: 3 of 3; 10 of 10, one new; 9 of 9 (01) |
| 2 | The IPC contract | all pass, the registry's `stock_adjust` row equal to the contract's | **pass**: 7 of 7 (01) |
| 3 | Every `terminal` and `pos-db` test; clippy | all pass; clean | **pass**: 485 of 485; clippy exit 0 (01) |
| 4 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `d98c0ed`: 1019 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog and the frontier reconcile (01) |
| 5 | C-100 reads the oil's stock and sells one bottle | ten on hand at 0.100, then nine; the sale's event captures 0.100 | **pass**: 10.000, then 9.000; `sale -1.000 · cost 0.100` (02) |
| 6 | M-100 asks to write one off as damaged, in a new process | the request stored, nothing moved | **pass**: request #1 stored; 9.000 on hand; 1 stored (02) |
| 7 | After a restart the owner approves exactly that request, and M-100 posts it | eight on hand; the sale's event unchanged; the audit row, the consumption, one envelope; the request gone, and not posted twice | **pass**: 8.000; `sale -1.000 · cost 0.100`; `stock.adjust` by M-100, approved by O-100, approval #1, reason "synthetic shelf check", `reason_code` damage; all three checks pass; 0 stored; then `stock_adjustment_unknown` (02) |
| 8 | Two bottles found by a count, approved by M-200 | ten on hand, at the cost in force, marked estimated | **pass**: 10.000 · 0.100 · estimated; the event's cost `0.100 (estimated)` (02) |
| 9 | A cashier reads stock and asks; a manager's requests no approval could make right | the cashier reads, bread's cost unknown, an unknown product refused, asking refused; each request refused by name before it is stored | **pass**: 10.000; bread `unknown · estimated`; `product_unknown`; `permission_denied`; `stock_reason_unknown`, `stock_adjustment_against_reason` three times with its detail, `product_unknown`, `qty_not_whole` (03) |
| 10 | Approvals of anything but exactly the request; a request changed by hand; a request someone else asked for | each refused, and nothing moves | **pass**: `approval_mismatch` for another request, another amount and someone else; the database refuses changing an approved request; a changed unapproved one is `stock_adjustment_altered`; `stock_adjustment_not_yours` (03) |
| 11 | M-100 posts request #1, then tries its approval on request #2 | posted once; `approval_used`; every other approval unspent; one bottle gone | **pass**: 9.000; `approval_used`; approvals #2–#4 unspent; 2 stored (03) |
| 12 | No shift open | refused, nothing spent | **pass**: `shift_not_open`; approval #1 unspent; 1 stored; 10.000 (04) |
| 13 | 17 planted defects | each turns the test that names its rule red | **pass**: 17 of 17 caught, each compiled (05) |

## Not validated here, and why

- **The screen.** `StockAdjust.tsx` and its test wait for 1.11.3b's design direction, so the last part of the `Done when` (`pnpm --filter terminal exec vitest run src/screens/StockAdjust.test.tsx`) does not exist yet.
- **Issuing an approval at the counter.** `auth_verify_pin` and the approval modal are 1.11.4b's. Until then no shipped command issues an approval, and the harness stores each one the way that screen will.
- **A clock-dated adjustment.** The register trusts no clock yet, so every adjustment is dated by its open shift; there is no trusted-clock branch to show.
