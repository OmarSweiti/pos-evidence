# Validation: the owner's decision D5 (1.10.5), a cashier is shown a product's stock, never its cost

**Commit validated:** `73c2b82`, the pull request's head. It was checked out clean in a scratch worktree with its own build directories.

**What changes at runtime:** one command's answer.
- **`stock_on_hand`'s `StockSnapshot` carries the cost as one field, `cost`.** It is `{ currency, wac_minor, is_wac_estimated }`, and it is `null` unless the signed-in user holds `stock.adjust`. A cost never recorded is shown to a holder as `wac_minor: null`, so a hidden cost and an unknown one stay apart.
- **What the user holds decides, not their role's name.** It is read from the grants they signed in with, as every proof is.
- **`stock_adjust` always answers the cost,** because only a holder of `stock.adjust` reaches it.

The runtime is shown through a harness linked against this head's `terminal`, `pos-db` and `pos-domain`.
- **Each step is a new process.** It opens the register with `Register::open_with`, as `run` opens it, on the shell suites' own shop file built by `pos_db::open`.
- **It drives the shipped commands:** sign-in, `on_hand`, `prepare_adjustment` and `adjust`, the plain functions the Tauri commands wrap.
- **A `WIRE` line is `stock_on_hand`'s answer serialized exactly as the screen receives it,** its product id replaced by the product's name.
- **The grant is made as an edit of `role_capability` would make it,** because no shipped command edits the grid yet.
- **No shipped command issues an approval yet** (`auth_verify_pin` is 1.11.4b's), so scenario C's approval is stored the way that screen will store it.
- **No PIN, hash or id is printed.**

**How it ran:**
- **The suites (01):** run in the worktree: 1.10.5's command tests, the IPC contract, every `terminal` and `pos-db` test, and clippy over `terminal` with `-D warnings`. Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–04):** built `--offline` against the worktree, with a fresh synthetic register file per scenario. The board builder refuses any of the following: a non-zero exit, an action whose outcome differs from the one written, the synthetic PIN, the word `argon2`, a UUID, or a local path. It also asserts every `WIRE` line the steps below name.
- **The planted defects (05):** 7 of them, run at this head before `just pr` against 1.10.5's command tests, each restored with git. The worktree was clean afterwards.

**Data is synthetic throughout:**
- **The shop:** the reference chain's fixture store, with register `REG01` and a shift open on it dated 2026-11-05.
- **The stock:** ten bottles of oil and half a kilogram of cheese posted as opening stock at 0.100 a unit, and no bread, whose cost was never recorded.
- **The people:** cashier C-100, shift lead L-100, managers M-100 and M-200, and the owner O-100, with one synthetic PIN.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | 1.10.5's command tests | all pass, the new one among them | **pass**: 10 of 10, `a_cashier_reads_stock_but_never_its_cost` among them (01) |
| 2 | The IPC contract; every `terminal` and `pos-db` test; clippy | all pass; clean | **pass**: 7 of 7; 495 of 495; clippy exit 0 (01) |
| 3 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `73c2b82`: 1030 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog and the frontier reconcile (01) |
| 4 | Cashier C-100 reads the oil, the cheese and the bread | what each shelf holds, and no cost; on the wire `cost: null` as a whole | **pass**: 10.000, 0.500, 0.000, each `cost: not shown`; `{"cost":null,"on_hand_milli":10000,...}` (02) |
| 5 | Shift lead L-100 reads the same | no cost | **pass**: `cost: not shown`; `cost: null` (02) |
| 6 | Manager M-100 reads the same | the cost; the bread's as unknown, inside the cost | **pass**: 0.100 JOD observed for the oil and the cheese; bread `unknown · estimated`, `{"currency":"JOD","is_wac_estimated":true,"wac_minor":null}` (02) |
| 7 | Owner O-100 reads the oil | refused as a whole: the owner holds no `sale.create`, which predates D5 | **refused** as written: `permission_denied` (02) |
| 8 | The merchant grants the cashier role `stock.adjust` mid-session | the session keeps its grants: no cost; the next sign-in is shown it | **pass**: `cost: null` before and after the grant; next sign-in shows 0.100 JOD and the bread's unknown cost (03) |
| 9 | M-100 writes off one bottle with the owner's approval; then the cashier reads the oil | `stock_adjust` answers the cost; the cashier is shown nine and no cost | **pass**: `9.000 on hand · cost a unit 0.100 JOD`; then `9.000 on hand · cost: not shown`, `cost: null` (04) |
| 10 | 7 planted defects | each turns the test that names its rule red | **pass**: 7 of 7 caught, each compiled (05) |

## Not validated here, and why

- **No screen.** No screen calls `stock_on_hand` yet. 1.10.5's `StockAdjust.tsx` waits for 1.11.3b's design direction, so the answer is shown as the screen would receive it, in the `WIRE` lines.
- **No device.** Nothing here touches a printer, drawer or scanner, so no emulator was started.
- **The owner's refusal (step 7)** is recorded, not changed. The pull request's "Read before approving" asks whether the owner should read stock at the till.
