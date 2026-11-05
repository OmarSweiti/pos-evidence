# Validation: 1.8.2c, the shell's database, session and command context

**Commit validated:** `c341d4f`, the pull request's head, checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **The shell opens the register database once, at start-up,** and holds it as the till.
  - A refusal keeps the till closed with a fixed sentence naming why.
  - Start-up recovery closes sessions a crash left open, reclaims parked carts, and reconciles the stock cache.
- **Three commands sign staff in and out:** `session_state`, `auth_login_pin` and `auth_logout`.
- **`ipc::context::authorize`** is the one path from a webview call to a permitted command. It checks:
  - that someone is signed in;
  - the role's grants, read whole from the grid;
  - a stored approval, held to its binding;
  - that the user is still live.
- **`pos-db`** gains staff accounts by code, the `user_session` rows, and a role's grants and roles at a store.

No screen calls these commands yet; the lock screen is a later step. The runtime is shown here through the shell's own code: a harness linked against this head's `terminal` crate. Each step runs it as a new process, which opens the register with `Register::open_with` and wraps the result with `Till::from_result`, as `run` does. It calls the real `login_pin`, `logout`, `state_of` and `authorize`.

**How it ran:**
- **The suites (01).**
  - The `Done when`, `cargo nextest run -p terminal --test session`, and the role grid's unit tests.
  - Then every terminal test, every `pos-db` test, and clippy over both crates.
  - Then the whole workspace, the test catalog and the frontier check.
- **The harness (02–06).**
  - Built `--offline` from the worktree's `Cargo.lock`, against a fresh synthetic register file per scenario.
  - Every step is its own process, and its exit status is recorded beside its output.
  - The clock is set per step, so the attempt policy's waits are shown at exact times.
  - The crash is a real `abort()`: no destructor runs and nothing is signed out.
  - The database key is a synthetic test key passed to `open_with`, so the OS credential store is not touched.
- **The planted defects (07).** 21 of them, each applied to the worktree, tested against the session suite and the role tests, then restored. No migration was mutated.

Data is synthetic throughout:
- an invented organisation;
- two stores (`FIX-S1`, `FIX-S2`);
- register `REG01`;
- six staff named "Synthetic …" with numbers `C-100` to `D-600`.

A PIN appears only as `right` or `wrong`. The board builder refuses to render any board holding a fixture PIN, a PIN hash, the key or a local path.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal --test session` | 14 pass | **pass**: 14 of 14 (01) |
| 2 | The role tests, terminal, `pos-db`, clippy, the workspace, the catalog, the frontier | all pass | **pass**: 3 of 3; 55 of 55; 317 of 317; clippy clean; 889 passed, 4 skipped (the existing `#[ignore]` ones); both checks reconcile (01) |
| 3 | No device id, a blank one, one no register carries, another key, a retired register, a file another build altered | the till stays closed and says which; a command answers `till_closed` with the same sentence | **pass**: six closed tills, each with its own reason; `login` refused `till_closed` with it (02) |
| 4 | This install's own device id and key | the till opens, recovery runs, and nobody is signed in | **pass**: `opened register REG01`, 0 sessions closed, 0 carts reclaimed, stock cache equal to the ledger; `signed_out` (02) |
| 5 | A wrong PIN, an unknown number, no role here, a role only at the other store, two roles | one `auth_refused` for the first two; the role cases refused by name after a correct PIN | **pass**: `auth_refused` twice; `auth_role_unresolved` "no role at this store" twice and "more than one role at this store" once (03) |
| 6 | Only a wrong PIN against a live account is counted and audited | C-100 counted; Z-999 leaves no row | **pass**: one `auth.login_refused` with `failed_attempts 1` for C-100; the role refusals are audited with `roles` 0, 0 and 2; nothing for Z-999 (03) |
| 7 | The same role held org-wide and at the store; then a cashier signed in | one role; may sell, may not administer users; a second sign-in refused | **pass**: D-600 signs in as `cashier`; C-100 `sale.create` allowed, `user.admin` `permission_denied`; M-200 `auth_already_signed_in` (03) |
| 8 | What the webview sees | staff number, name and role only | **pass**: every `session_state` and sign-in answer carries `state`, `code`, `display_name`, `role` and nothing else (03, 04, 05, 06) |
| 9 | Four wrong PINs, then the right one at once | the first three free; the fourth starts a 30 s wait; the right PIN not checked inside it | **pass**: `auth_delayed`; "no PIN checked before 09:00:30" (04) |
| 10 | Wrong PINs at each wait's end | the wait doubles to at most 15 minutes; the tenth locks the user out | **pass**: allowed again at 09:01:30, 09:03:30, 09:07:30, 09:15:30, then 09:30:30 (capped); the tenth refused `auth_locked_out`, audited with `locked_out: true` (04) |
| 11 | The right PIN an hour later, then after a release | refused until released; then signed in, count reset | **pass**: `auth_locked_out` at 10:30; after `UserRepository::release`, signed in at 10:31; "0 in a row" (04) |
| 12 | A command with no one signed in; the cashier without and with a manager's approval | `not_signed_in`; `approval_required`; allowed for 1.250 JOD with the approval, approved by M-200 | **pass**: as expected (05) |
| 13 | The same approval for 1.251 JOD; an id no handle has; the approval after it is consumed | each refused by name | **pass**: `approval_mismatch` "the approval is for another amount"; `approval_unknown`; `approval_used` (05) |
| 14 | C-100 deactivated mid-session | signed out before acting; cannot sign in again | **pass**: `user_inactive`; `signed_out`; `auth.logout` with cause `user_inactive`; then `auth_refused` (05) |
| 15 | A signed-in process aborts; the next process starts | the open session is closed at start-up as a crash; the till is signed out | **pass**: exit 134; the file holds an open session; next start-up "1 session(s) left open by a crash closed", ended 09:42:00 `crash`; `signed_out` (06) |
| 16 | 21 planted defects | each turns the test that names its rule red | **pass**: 21 of 21 caught; every mutant compiled, and the worktree was restored with no changed file (07) |

`just pre-push` was run by `just pr` before the pull request was opened, at `e53902c`, and passed: lint, tests, the web build, guards and the secret scan. The head since then, `c341d4f`, changes only the session suite: it adds a closed-account case and stops a failing assertion from printing a fixture PIN, which CodeQL had flagged. Steps 1, 2 and 16 ran at that head, and every required CI check is green on it.

## Not validated, and why

- **A lock screen signing someone in.** No screen calls these commands yet, so the app was not launched. The commands are shown through the functions the Tauri handlers wrap.
- **Opening with the key from the OS credential store.** `Register::open` reads the device id first, then the key from the credential store. The harness passes a synthetic key to `open_with`, the same function with the key supplied, so the credential store is never touched.
- **A fresh install with no device id opens closed.** This is expected until 1.8.5b's provisioning writes the device id. The pull request and the workflow document (§17) name the gap.
- **The discount and its approval's consumption are the harness's.** The context's resolution, binding and `approval_used` refusal are the shell's. The `discount.manual` command that consumes an approval beside its audit row belongs to a later step, so the harness performs that consumption as the command will.
- **The audit rows' delivery envelopes are not printed.** The database's `audit_log_has_ready_commit` trigger refuses any audit row without a ready envelope, so each row shown committed with one.
