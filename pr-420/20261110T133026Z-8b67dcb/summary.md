# Validation: the owner's decision D1 B (1.6.2), one manager resets a PIN alone, with a reason code

**Commit validated:** `8b67dcb`, the pull request's head. It was checked out clean in a scratch worktree with its own build directories.

**What changes at runtime:** `user_reset_pin { user_id, new_pin, reason, approval_id? }`.
- **One person is enough by default.** A holder of `user.admin` resets alone, under their own PIN, and the audit row names them, the account and the reason, and no second person.
- **The reason is a code:** `forgot_pin`, `locked_out` or `pin_disclosed`. Anything else is `pin_reset_reason_unknown` and writes nothing.
- **A second person only where the merchant's policy names `user.admin`.** The registry row is `Conditional` on `ApprovalPredicate::UserAdminEscalatedByPolicy`. The command passes `merchant_policy()`, which names nothing yet. Under a policy that does, the approval must name this account, exactly zero and the same reason, and it is spent with the audit row.
- **Unchanged:** a store's manager may not reset the owner's PIN (D2), and every account under the shipped grid needs six digits (D3).

The runtime is shown through a harness linked against this head's `terminal`, `pos-db` and `pos-domain`.
- **Each step is a new process.** It opens the register with `Register::open_with`, as `run` opens it, on a file built by `pos_db::open` from the reference chain's fixture.
- **It drives the shipped commands:** sign-in and `reset_pin`, the plain function the Tauri command wraps.
- **`reset` passes `merchant_policy()`,** the policy the Tauri command passes. `reset-escalated` passes a policy that names `user.admin`, as a merchant's could; no merchant's policy is stored yet.
- **No shipped command issues an approval yet** (`auth_verify_pin` is 1.11.4b's), so `approve` stores one as that screen will: issued from the approver's own proof of `user.admin`, behind its envelope.
- **No PIN, hash or id is printed.** The board says which PIN was used, never its digits.

**How it ran:**
- **The suites (01):** run in the worktree:
  - 1.6.2's `Done when`: `cargo nextest run -p pos-db auth::`, `cargo nextest run -p terminal manager_reset_`, and the `pin_verify` benchmark run once as a test;
  - the reset suite and the IPC contract;
  - every `terminal` and `pos-db` test;
  - clippy over both with `-D warnings`.

  Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–05):** built `--offline` against the worktree, with a fresh synthetic register file per scenario. The board builder refuses any of the following: a non-zero exit, an action whose outcome differs from the one written, a failed check, either synthetic PIN, the word `argon2`, a UUID, or a local path. It also asserts the key lines the steps below name.
- **The planted defects (06):** 10 of them, run at this head before `just pr` against the reset suite and the IPC contract, each restored with git. The worktree was clean afterwards.

**Data is synthetic throughout:** the reference chain's fixture store with registers REG01 and REG02; cashiers C-100 and C-200, managers M-100 and M-200, the owner O-100, and C-300, an account with no role. Every account starts with one synthetic six-digit PIN.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | 1.6.2's `Done when` | the auth tests, E.51 and the benchmark run once pass | **pass**: 13 of 13; `manager_reset_retires_old_hash_and_audits`; both benchmark cases `Success` (01) |
| 2 | The reset suite and the IPC contract | all pass, the new test and the predicate's boundary among them | **pass**: 11 of 11, `a_lone_reset_names_its_reason_from_the_list` and `conditional_privilege_cannot_cross_threshold_without_approval` among them (01) |
| 3 | Every `terminal` and `pos-db` test; clippy | all pass; clean | **pass**: 496 of 496; clippy exit 0 (01) |
| 4 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `8b67dcb`: 1031 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog and the frontier reconcile (01) |
| 5 | C-100 locks themselves out; M-100 resets them alone for `locked_out` | no second person asked for; a four-digit PIN for a manager refused; the old PIN retired, the lockout on REG01 released, C-200's attempts kept; the audit row names M-100, no second person and the reason | **pass**: `pin_shape · this account needs six digits or more`; reset alone; C-100 `0` on REG01, C-200 `2`; old `false`, new `true`; `by M-100 · no second person · app_user C-100 · reason "locked_out"`; C-100 then signs in with the new PIN only (02) |
| 6 | Reasons not on the list | each refused, nothing written | **pass**: free text, `Forgot_pin`, `shrinkage` and a blank each `pin_reset_reason_unknown`; PIN unchanged; no audit row (03) |
| 7 | The three codes, an approval offered with the third | each recorded as given; the offered approval unspent | **pass**: `forgot_pin`, `locked_out`, `pin_disclosed`, each `no second person`; `approval #1: unspent` (03) |
| 8 | Under a policy that names `user.admin`: no approval, and approvals for another reason, account or amount | `approval_required`; each `approval_mismatch` with its detail, unspent, PIN unchanged, no audit row | **pass**: `approval_required`; `the approval was given for another reason`, `the approval names something else`, `the approval is for another amount`; #1–#3 unspent; no audit row (04) |
| 9 | The owner's approval of exactly this reset | the reset passes; the audit row names the approver and approval, spent once | **pass**: `approved by O-100 · approval #4`; spent beside the row; again `approval_used` (04) |
| 10 | A cashier resets; M-100 resets the owner, alone and with the deputy's approval; a deactivated and a roleless account; the owner resets M-100 | refused; refused both ways, approval unspent; refused; passes alone | **pass**: `permission_denied`; `permission_outside_limit` twice, approval #1 unspent, the owner's PIN unchanged; `user_unknown` twice; `by O-100 · no second person · app_user M-100` (05) |
| 11 | 10 planted defects | each turns the test that names its rule red | **pass**: 10 of 10 caught, each compiled (06) |

## Not validated here, and why

- **No screen.** No screen calls `user_reset_pin` yet. Its screen waits for 1.11.3b's design direction, so the shipped command's plain function is driven directly.
- **No merchant's policy is stored,** so the escalated path runs under a policy built in the harness, the same type the command takes.
- **No daily exception report exists yet.** Merchant decision 9.6's default lists a reset in one; no step builds it, so the audit rows are shown instead.
- **No device.** Nothing here touches a printer, drawer or scanner, so no emulator was started.
