# Validation: 1.6.2's PIN reset, a locked-out cashier reset by a manager

**Commit validated:** `5072b9c`, the pull request's head. It was checked out clean in a scratch worktree with its own build directories.

**What changes at runtime:** a new command, `user_reset_pin { user_id, new_pin, approval_id }`.
- **Every call takes `user.admin` and a bound approval.** The approval names the account being reset, exactly zero, and its approver's reason, which the audit row repeats as it spends the approval. The approver is never the person resetting.
- **A store's administrator does not reset the organisation's.** A holder of `user.admin` limited to their own store cannot reset an account whose `user.admin` has no limit, such as the owner's.
- **The new PIN has the digits its account's grants need.**
- **The old hash is retired, and the account's lockout on this register released,** with the audit row and the approval's consumption, in one transaction.

The runtime is shown through a harness linked against this head's `terminal`, `pos-db` and `pos-domain`.
- **Each step is a new process.** It opens the register with `Register::open_with` on a file built by `pos_db::open`, as `run` opens it.
- **It drives the shipped sign-in and `reset_pin`,** the plain functions the Tauri commands wrap.
- **The lockout is reached through the shipped sign-in.** Its clock moves past each wait sign-in imposes, until the tenth wrong PIN locks the account out.
- **No shipped command issues an approval yet:** `auth_verify_pin` is 1.11.4b's. So `approve` stores one as that screen will: issued from the approver's own proof of `user.admin`, for the signed-in user, behind its envelope.
- **No PIN, hash or id is printed.** A PIN is named only as old, new, wrong or four digits; an approval by the order it was stored in.

**How it ran:**
- **The suites (01):** run in the worktree:
  - 1.6.2's `Done when`: `cargo nextest run -p pos-db auth::`, `cargo nextest run -p terminal manager_reset_`, and the `pin_verify` benchmark run once as a test;
  - the command's suite and the IPC contract;
  - every `terminal` and `pos-db` test;
  - clippy over both crates with `-D warnings`.

  Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–03):**
  - Built `--offline` against the worktree, with a fresh synthetic register file per scenario.
  - The board builder refuses a non-zero exit, an action whose outcome differs from the one written, a failed check, any of the synthetic PINs, the word `argon2`, a UUID or a local path.
- **The planted defects (04):** 13 of them, run at this head before `just pr`.
  - Each was run against the suite its rule belongs to, then restored with git.
  - The worktree was clean afterwards.
- **Builds ran without debug information** (`CARGO_PROFILE_DEV_DEBUG=0`).

**Data is synthetic throughout:**
- the reference chain's fixture merchant and its store, with registers `REG01` and `REG02`;
- cashiers C-100 and C-200, managers M-100 and M-200, the owner O-100, and C-300, an account with no role;
- one synthetic six-digit PIN that every account starts with.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | 1.6.2's `Done when` | the database half's tests, the named reset test, and the benchmark run once | **pass**: 13 of 13; `manager_reset_retires_old_hash_and_audits`; `pin_verify/right` and `/wrong` (01) |
| 2 | The command's suite and the IPC contract | all pass | **pass**: 10 of 10 (01) |
| 3 | Every `terminal` and `pos-db` test; clippy | all pass; clean | **pass**: 475 of 475; clippy exit 0 (01) |
| 4 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `5072b9c`: 1009 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog and the frontier reconcile (01) |
| 5 | C-100 tries wrong PINs through the shipped sign-in until locked out; the process restarts | three free, then each waits; the tenth locks out; a restart releases nothing | **pass**: wrong PINs 1–9 `auth_refused`, each from the fourth followed by `auth_delayed`; the tenth `auth_locked_out`; 10 counted on REG01; after the restart still `auth_locked_out` (02) |
| 6 | M-100 resets with the owner's approval: the deputy to four digits, C-200 under C-100's approval, then C-100, then again | a manager needs six digits; an approval for another account refused; the reset done; the approval spent once | **pass**: `pin_shape`, six digits; `approval_mismatch`, names something else; C-100 reset; then `approval_used` (02) |
| 7 | After the reset: the counts, the stored hash, the audit row; C-100 signs in | the lockout on REG01 released, C-200's two stand; the old PIN retired; the row names manager, owner, approval and reason, spends the approval, and holds neither PIN nor hash; the new PIN signs in and the old does not | **pass**: C-100 0 on REG01, C-200 2; old PIN false, new true; `user.admin` by M-100, approved by O-100, approval #2, `app_user` C-100, reason "forgot their PIN", `failed_attempts_cleared` 10; both checks pass; `auth_refused` with the old PIN, signed in with the new (02) |
| 8 | A cashier resets; M-100 resets the owner, a deactivated account and one with no role, and uses an approval for 10 fils | each refused by name | **pass**: `permission_denied`, `permission_outside_limit`, `user_unknown` twice, `approval_mismatch`, for another amount (03) |
| 9 | The owner resets M-100 with M-200's approval | the reset done, audited | **pass**: M-100's old PIN false, new true; `user.admin` by O-100, approved by M-200 (03) |
| 10 | 13 planted defects | each turns the test that names its rule red | **pass**: 13 of 13 caught, each compiled (04) |

## A required check that is red, and why

**`protected-paths` is red by design.** The byte-pinned `scripts/check-test-catalog.py` moves one name, `manager_reset_retires_old_hash_and_audits`, from `PLANNED` to `PLANNED_RETIRED`, and needs the owner's read.

## Not validated here, and why

- **Issuing an approval at the counter.** `auth_verify_pin` and the approval modal are 1.11.4b's. Until then no shipped command issues one, and the harness stores each as that screen will.
- **A second factor beyond a manager's PIN.** #233 is the merchant's to answer; this ships under merchant decision 9.6's default.
- **The benchmark's budget.** The 200–350 ms band on the reference register is 5.8.2's; the benchmark here only runs once.
