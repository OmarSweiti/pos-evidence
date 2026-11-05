# Validation: 1.6.4's last item, an approver's deactivation revokes their approvals

**Commit validated:** `c8da3eb`, the pull request's head, checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **The command context checks the approver.** Once the domain has accepted an approval for a command, the context reads the approver's account.
- **A deactivated or deleted approver's approval is refused** as `approval_revoked`, with the detail "the approver's account is no longer active".
- **Nothing else changes.** Nothing is written, the approval stays unspent, the cashier stays signed in, and a command that doesn't escalate ignores the approval id.
- **The domain gains `PermissionError::ApproverInactive`** for the refusal.

No screen calls the context yet. As for 1.8.2c, the runtime is shown through a harness linked against this head's `terminal` crate:
- Each step runs it as a new process, which opens the register as `run` does.
- It calls the context's own `authorize`, `login_pin` and `logout`.

**How it ran:**
- **The suites (01).**
  - 1.6.4's `Done when`, its four suites.
  - Every terminal test, and clippy over `terminal` and `pos-domain` with `-D warnings`.
  - The test catalog and the frontier check.
  - Then `just pr`'s own gate at this head, from the main checkout: the whole workspace, lint, the web build and tests, the guards and the secret scan.
- **The scenario (02).**
  - One synthetic register file, three processes.
  - The clock is set per step, so the approval's sixty-second life is visible.
  - A deactivation is written to `app_user` the way a deactivation synced from the back office arrives.
  - The database key is a synthetic test key passed to `open_with`, so the OS credential store is not touched.
- **The planted defects (03).** Seven, each applied to the worktree, tested against the session suite, then restored. No migration was mutated.

Data is synthetic throughout: an invented store and register, and staff named "Synthetic …" with numbers `C-100` and `M-200`. A PIN appears only as `right`.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | 1.6.4's `Done when` | all four suites pass | **pass**: session 15 of 15, including `an_approval_is_revoked_when_its_approver_is_deactivated`; `permissions::` 18 of 18; `typed_ids_ui` 3 of 3; `approval` 5 of 5 (01) |
| 2 | Every terminal test, clippy, the catalog, the frontier, and `just pr`'s gate | all pass | **pass**: terminal 56 of 56; clippy clean; both checks reconcile; the gate's workspace 890 passed, 4 skipped (the existing `#[ignore]` ones), web tests and build, guards and secret scan all passed (01) |
| 3 | A manager's approval, used while their account is live | allowed, approved by M-200 | **pass**: allowed at 09:00:10, approved by M-200 (02) |
| 4 | The same approval after M-200 is deactivated | `approval_revoked`, naming why | **pass**: refused at 09:00:20, "the approver's account is no longer active" (02) |
| 5 | The cashier, after the refusal | still signed in; a command needing no approval runs; an unescalated discount ignores the id | **pass**: `signed_in` as C-100; `sale.create` allowed; the unescalated discount allowed with "no approval used" (02) |
| 6 | Nothing written by the refusal | the approval not spent; no sign-out for it | **pass**: the approval is "not spent"; the audit log holds only the requested sign-in and sign-out (02) |
| 7 | After a restart, the stored approval passed by its id again | still refused | **pass**: `approval_revoked` at 09:00:30 in a new process (02) |
| 8 | M-200 made live again, then deleted instead of deactivated | allowed while live; refused once deleted | **pass**: allowed, approved by M-200; then `approval_revoked` with `deleted_at` set and `is_active` still 1 (02) |
| 9 | The approval at its expiry, 09:01:00 | refused as expired, by the domain, before the approver is read | **pass**: `approval_expired` (02) |
| 10 | Seven planted defects | each turns the test that names its rule red | **pass**: 7 of 7 caught; every mutant compiled, and the worktree was restored with no changed file (03) |

## Not validated, and why

- **A screen.** No screen calls the context yet, so the app was not launched. The context is shown through the function every command will call.
- **A deactivation arriving by sync.** Sync pull doesn't exist yet. The harness writes `app_user` as a synced deactivation would, and the context reads only the account's current state.
- **The finalize re-checking in its transaction.** 1.8.3's finalize doesn't exist yet. Its section now says its gate must read the approver again, by the same rule, before the financial write.
- **An approver with no account.** This can't be set up: `approval_handle.approver_id` is a foreign key to `app_user`. The code refuses that case anyway.
