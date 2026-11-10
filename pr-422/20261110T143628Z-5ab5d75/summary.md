# Validation: the owner's decision D1 B (1.10.5), one manager adjusts stock at once with stock_adjust_now

**Commit validated:** `5ab5d75`, the pull request's head. It was checked out clean in a scratch worktree with its own build directories.

**What changes at runtime:** a new command, `stock_adjust_now { product_id, qty_delta_milli, reason_code, note? } -> StockSnapshot`, under `stock.adjust`, audited, taking no approval.
- **It posts the `adjust` event an approved request would,** through the same ledger path: by the person signed in, naming its reason and note, dated by the open shift. No request is stored, and no approval is spent.
- **The event and its `stock.adjust` audit row travel in one envelope,** the event first. The row names no second person, and its reason is the reason code.
- **Where a merchant's policy names `stock.adjust`, it is `approval_required`,** and the two-step path, unchanged, is the way.

The runtime is shown through a harness linked against this head's `terminal`, `pos-db` and `pos-domain`.
- **Each step is a new process.** It opens the register with `Register::open_with`, as `run` opens it, on the shell suites' own shop file built by `pos_db::open`.
- **It drives the shipped commands:** sign-in, `on_hand`, `adjust_now`, `prepare_adjustment` and `adjust`, the plain functions the Tauri commands wrap.
- **`now` passes `merchant_policy()`,** the policy the Tauri command passes. `now-escalated` passes a policy that names `stock.adjust`, as a merchant's could; none is stored yet.
- **No shipped command issues an approval yet** (`auth_verify_pin` is 1.11.4b's), so scenario B's approval is stored as that screen will store it.
- **No PIN, hash or id is printed.** Direct events are named direct #n, requests and approvals by order.

**How it ran:**
- **The suites (01):** run in the worktree: 1.10.5's `Done when` but its screen, the IPC contract, every `terminal` and `pos-db` test, and clippy over `terminal` with `-D warnings`. Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–05):** built `--offline` against the worktree, with a fresh synthetic register file per scenario. The board builder refuses any of the following: a non-zero exit, an action whose outcome differs from the one written, a failed check, the synthetic PIN, the word `argon2`, a UUID, or a local path. It also asserts the key lines the steps below name.
- **The planted defects (06):** 9 of them, run at this head before `just pr`, each restored with git. The worktree was clean afterwards.
  - A first run at the previous head planted a tenth: removing the command's own check of the reason's direction. No test turned red. That was correct: the domain's `StockEvent::capture` refuses the same movement with the same error before anything is written. So the check was removed in this pull request, and the defect was replaced by one that names who posted.

**Data is synthetic throughout:**
- **The shop:** the reference chain's fixture store, with register `REG01` and a shift open dated 2026-11-05 (none in scenario D).
- **The stock:** ten bottles of oil and half a kilogram of cheese posted as opening stock at 0.100 a unit.
- **The people:** cashier C-100, shift lead L-100, managers M-100 and M-200, and the owner O-100, with one synthetic PIN.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | 1.10.5's `Done when`, but its screen | the opening-stock tests, the database suite and the commands' suite pass | **pass**: 3 of 3; 10 of 10; 13 of 13, the three new ones among them (01) |
| 2 | The IPC contract; every `terminal` and `pos-db` test; clippy | all pass; clean | **pass**: 7 of 7; 499 of 499; clippy exit 0 (01) |
| 3 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `5ab5d75`: 1034 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog and the frontier reconcile (01) |
| 4 | M-100 writes off one bottle at once | nine on hand at the cost in force; no request stored; the event names the reason, note, manager and the shift's day; the audit row names no second person and the reason code; no approval spent; one envelope, the event then the audit row | **pass**: 9.000 · 0.100 · observed; 0 stored; `adjust -1.000 · damage, note "one bottle dropped" · by M-100 · 2026-11-05`; `by M-100 · no second person · reason "damage"`; all three checks pass (02) |
| 5 | M-100 finds two bottles by a count; the cashier reads the oil | eleven on hand, at the cost in force, estimated; the cashier sees no cost | **pass**: 11.000 · 0.100 · estimated, the event's cost `0.100 (estimated)`; `11.000 on hand · cost: not shown` (02) |
| 6 | Under a policy that names `stock.adjust`, M-100 adjusts at once | refused; nothing stored or moved | **pass**: `approval_required`; 0 stored; 10.000; only opening stock in the ledger (03) |
| 7 | The two steps under that policy | the request posts with the owner's approval | **pass**: request #1 posted under approval #1, 9.000; the audit row names O-100 and approval #1, spent beside it (03) |
| 8 | A cashier, then a manager's adjustments no rule allows | each refused by name; nothing stored, posted or audited | **pass**: `permission_denied`; `stock_reason_unknown`; `stock_adjustment_against_reason` three times with its detail; `product_unknown`; `qty_not_whole`; 0 stored, 10.000, only opening stock, no `stock.adjust` row (04) |
| 9 | No shift open | refused, nothing moved | **pass**: `shift_not_open`; 10.000; only opening stock (05) |
| 10 | 9 planted defects | each turns the test that names its rule red | **pass**: 9 of 9 caught, each compiled (06) |

## Not validated here, and why

- **No screen.** No screen calls `stock_adjust_now` yet. 1.10.5's `StockAdjust.tsx` waits for 1.11.3b's design direction, so the shipped command's plain function is driven directly.
- **No merchant's policy is stored,** so the escalated path runs under a policy built in the harness, the same type the command takes.
- **No device.** Nothing here touches a printer, drawer or scanner, so no emulator was started.
