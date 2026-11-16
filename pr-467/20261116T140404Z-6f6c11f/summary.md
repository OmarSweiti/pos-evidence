# Validation: 1.11.4b part one, `auth_verify_pin` issues a bound approval

**Commit validated:** `6f6c11f`, the pull request's head, in the repository's checkout with no change in the tree.

**What changes at runtime:** a new command, `auth_verify_pin { user_code, pin, for_capability, entity_id, amount_minor, reason } -> ApprovalRef`.
- **The approver is checked as a sign-in is.** Sign-in's PIN check is now one function, which `auth_login_pin` and the approval share. A wrong PIN counts on the approver's one counter and is audited as `auth.approval_refused`.
- **The approval is bound to the operation as its spending command checks it,** read from that command's registry row. For a prepared stock adjustment, the shell reads the request as stored.
- **It is stored for two minutes,** behind its envelope, with an `approval.issue` audit row by the approver. The webview is told only `ApprovalRef { approval_id, capability, expires_at }`.

There is no screen yet: the modal is part two. The runtime is shown through a harness linked against this head's `terminal`, `pos-db` and `pos-domain`.
- **Each step is a new process.** It opens the register with `Register::open_with`, on the shell suites' own shop file, as `run` opens it.
- **It drives the shipped functions** that the Tauri commands wrap: sign-in, sign-out, `approve` (`auth_verify_pin`), `reset_pin`, `prepare_adjustment`, `adjust`, `open_cart`, `add_product` and `override_line_price`.
- **Every approval is issued by `approve`.** Nothing is stored by hand.
- **No PIN, hash, nonce, digest or id is printed.** Requests, lines and approvals are named by the order they were made in.

**How it ran:**
- **The suites (01):** run in the checkout:
  - `cargo nextest run -p terminal --test approval`;
  - the IPC contract and the suites whose commands spend an approval;
  - the stock unit tests and the IPC type drift tests;
  - every `terminal` and `pos-db` test;
  - clippy over both crates with `-D warnings`.

  Then `just pr`'s gate at this same head is quoted from its log. Its one `LEAK` line, `auth::tests::argon2_parameters_match_the_reviewed_profile`, passed. nextest flags it as leaky: it is a `pos-db` test this pull request does not touch.
- **The harness (02–06):**
  - Built `--offline` against the checkout, with a fresh synthetic register file per scenario.
  - The board builder refuses a non-zero exit, an action whose outcome differs from the one written, a failed check, any synthetic PIN, an argon2 hash prefix, a UUID, a long hex run, or a local path.
- **The planted defects (07):** 12 of them, run on the committed tree at `6f6c11f` before `just pr`.
  - Each was run against its one test, then restored with git.
  - The tree was clean afterwards.

**Data is synthetic throughout:**
- the shell suites' fixture store, with register `REG01`, a shift open on 2026-11-05, and ten bottles of oil at 1.250;
- cashier C-100, shift lead L-100, managers M-100 and M-200, and the owner O-100;
- one synthetic six-digit PIN for every account.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 01 | The suites and the gate at `6f6c11f` | All green | `--test approval` 10 passed. The spending suites 48 passed, the stock and type tests 16, and `terminal` with `pos-db` 605. Clippy exit 0. The gate: 1149 Rust tests passed (1 leaky, named above), 4 skipped; web, guards and secrets clean. |
| 02 | A PIN reset approved at the counter, where the policy names `user.admin` | Nobody signed in, self-approval, an approver without `user.admin`, another amount, a capability no command spends, and a blank reason are each refused, storing nothing. The deputy's approval is spent once. | `not_signed_in`, `approval_self`, `permission_denied`, `approval_mismatch`, `approval_capability_unknown`, `approval_reason_missing`. Approval #1 was told as its three keys, lapsing at 09:02:00, stored behind its envelope. The reset spent it, and a second use was `approval_used`. The audit shows `approval.issue` by M-200 and `user.admin` by M-100 spending #1, with no PIN or hash in any row. |
| 03 | A stock adjustment approved as its request is stored | Another amount, another reason, an unknown request and an approver without `stock.adjust` are refused. The approval carries the digest, is refused on another request, and posts its own once. | `approval_mismatch` (amount), `approval_mismatch` (reason), `stock_adjustment_unknown`, `permission_denied`. Approval #1 has its digest bound. On request #2 it was `approval_mismatch`. On request #1, oil on hand went from 10 to 9, and a second post was `stock_adjustment_unknown`. |
| 04 | A price override approved for one price (the step's `Done when`, register side) | Approved at 1.000; refused at 0.900 and left unspent; spent at 1.000; then used | `approval_mismatch · the approval is for another amount`, still unspent. Then line #1 was set to 1.000 under #1, and the next use was `approval_used`. |
| 05 | One counter and one lockout | Two wrong PINs at the approval and two at sign-in make four; the right PIN waits at both. After the wait, approval works and the count resets. Ten wrong at the approval lock the owner out of both. | Four counted, then `auth_delayed` at both. After 31 s, approval #1 was issued and the count was 0. The owner's tenth wrong PIN was `auth_locked_out`, and then both the approval and sign-in answered `auth_locked_out`. Audit: `auth.approval_refused` × 2 for M-200 and × 10 for O-100. |
| 06 | Two minutes, across processes | Spent at 09:01:59.999; refused at 09:02:00.000 | A new process at 09:01:59.999 reset under #1. Another at 09:02:00.000 was refused #2 with `approval_expired`, and #2 stayed unspent. |
| 07 | Planted defects | Each caught by its test | 12 of 12 caught. |

## Not validated here

- **The modal and its screen,** which are part two (1.11.4b's `Done when`).
- **The Tauri IPC round trip itself.** The harness calls the plain functions the commands wrap, and the `ipc_contract` suite holds the registered command's argument schema.
- **No emulator is involved:** nothing here touches a device.
