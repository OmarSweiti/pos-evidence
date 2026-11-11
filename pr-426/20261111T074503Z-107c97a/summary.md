# Validation: the owner reads stock at the till (1.10.5, the owner's decision of 11 November 2026)

**Commit validated:** `107c97a`, the pull request's head. It was checked out clean in a scratch worktree with its own build directories.

**What changes at runtime:** who may call `stock_on_hand`.
- **It is registered under `sale.create` or `stock.adjust`,** through a new registry form, `Access::AnyOf`. The handler proves `sale.create`, or else `stock.adjust`.
- **Under the shipped grid only the owner gains it:** the owner holds `stock.adjust` and not `sale.create`. A user holding neither is `permission_denied`, as before.
- **The cost rule is unchanged:** only a holder of `stock.adjust` is shown it.
- **`check` refuses three misuses of `AnyOf`:** an alternative that is not a capability, fewer than two alternatives, and alternatives on a command that takes an approval.

The runtime is shown through a harness linked against this head's `terminal`, `pos-db` and `pos-domain`.
- **Each step is a new process.** It opens the register with `Register::open_with`, as `run` opens it, on the shell suites' own shop file built by `pos_db::open`.
- **It drives the shipped commands:** sign-in, `on_hand` and `adjust_now`, the plain functions the Tauri commands wrap.
- **A `WIRE` line is `stock_on_hand`'s answer serialized as the screen receives it,** its product id replaced by the product's name.
- **`withhold` edits `role_capability` as a merchant's edit would,** because no shipped command edits the grid yet.
- **No PIN, hash or id is printed.**

**How it ran:**
- **The suites (01):** run in the worktree: 1.10.5's command tests, the IPC contract, every `terminal` and `pos-db` test, and clippy over `terminal` with `-D warnings`. Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–03):** built `--offline` against the worktree, with a fresh synthetic register file per scenario. The board builder refuses any of the following: a non-zero exit, an action whose outcome differs from the one written, the synthetic PIN, the word `argon2`, a UUID, or a local path. It also asserts the key lines the steps below name.
- **The planted defects (04):** 7 of them, run at this head before `just pr`, each restored with git. The worktree was clean afterwards.

**Data is synthetic throughout:**
- **The shop:** the reference chain's fixture store, with register `REG01` and a shift open dated 2026-11-05.
- **The stock:** ten bottles of oil and half a kilogram of cheese posted as opening stock at 0.100 a unit, and no bread.
- **The people:** cashier C-100, shift lead L-100, managers M-100 and M-200, and the owner O-100, with one synthetic PIN.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | 1.10.5's command tests | all pass, the new one among them | **pass**: 14 of 14, `the_owner_reads_stock_and_one_holding_neither_capability_is_refused` among them (01) |
| 2 | The IPC contract; every `terminal` and `pos-db` test; clippy | all pass, `AnyOf`'s three refusals among them; clean | **pass**: 7 of 7; 500 of 500; clippy exit 0 (01) |
| 3 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `107c97a`: 1035 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog and the frontier reconcile (01) |
| 4 | The cashier, the shift lead and a manager read the oil | each reads; only the manager is shown the cost | **pass**: `10.000 · cost: not shown` twice, `cost: null` on the wire; the manager `0.100 JOD · observed` (02) |
| 5 | The owner reads the oil and the bread, writes off a bottle at once, and reads again | the owner reads, with the cost; the bread's as unknown; nine after the write-off | **pass**: `10.000 · 0.100 JOD`; bread `unknown · estimated`; posted at once; `9.000 · 0.100 JOD` (02) |
| 6 | The merchant withholds `sale.create` from the cashier role | the session keeps its grants; the next sign-in, holding neither capability, is refused; signed out, `not_signed_in` | **pass**: still reads after the edit; then `permission_denied`; then `not_signed_in` (03) |
| 7 | 7 planted defects | each turns the test that names its rule red | **pass**: 7 of 7 caught, each compiled (04) |

## Not validated here, and why

- **No screen.** No screen calls `stock_on_hand` yet. 1.10.5's screen waits for 1.11.3b's design direction, so the answer is shown as the screen would receive it.
- **No device.** Nothing here touches a printer, drawer or scanner, so no emulator was started.
