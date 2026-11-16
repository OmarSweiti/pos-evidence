# 1.11.4, part one: the generated IPC types and the lock screen's data — validation at 52a85cc

**Validated commit:** `52a85cc`, this pull request's head, in a checkout with no change to any tracked file, and a detached worktree at the same commit for the harness.

**Runtime:** what the register answers over IPC, as the webview receives it. There is no screen yet; that is part two.
- **The harness** is evidence-only, linked against this head's own `terminal` and `pos-db`. It drives a new machine through custody and store setup, then starts the till again as a new process.
- **It calls the commands' own functions:** `state_of` (`session_state`), `login_pin` (`auth_login_pin`) and `health` (`health_status`). Each answer is printed as serde JSON.
- **Each answer is also type-checked** with the repository's own `tsc` against the generated `packages/api-types/src/ipc/`.

**Data:** synthetic only: `pos_db::fixtures::store_setup`'s file (synthetic store, staff and products). The OS credential store is a stand-in file, never this Mac's Keychain. The recovery code goes to a paper file that is deleted after the run and never printed. No PIN, hash, key or code appears in any image, and a refusal's random trace id is masked.

**Emulators:** none; no device is touched by this change. The printer reads `no_printer`, as an unbound register does.

## Commands

- `cargo nextest run -p terminal --lib ipc_types_`, `--test lock_screen`, `--test session`, and `-p pos-db --test outbox`, at this head.
- Eleven planted defects, each one wrong edit run on the committed tree and reverted with `git checkout`.
- `just pr`'s gate (`just pre-push`) at this head; CI's required checks; `cargo deny check licenses bans sources`.
- The harness: `custody`, `file`, `apply`, then `lock <machine> 2026-11-16`.
- `tsc --noEmit --strict` on the saved answers, and on one deliberately broken copy.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | Tests and gate (01) | The part's tests and the suites it touched pass; the gate green | 27 passed, none failed: `ipc_types_` 2, `lock_screen` 2, `session` 15, `outbox` 8. Gate: 1139 workspace tests passed (4 skipped); vitest 10, 15, 66 and 3; both web builds, every guard suite, the secret scan. CI: every required check but `validation` passed. `cargo deny` OK; the lock adds exactly `ts-rs` and `ts-rs-macros`. |
| 2 | Planted defects (02) | Each turns red the test aimed at it | 11 of 11 caught: the waiting count's states either way; the open shift never told, named by staff number, or missing when signed in; the sync count zeroed; `bigint` left in; the `_key` rule dropped; a generated file edited by hand; a doc changed without regenerating; a PIN hash on a webview type. |
| 3 | Before anyone signs in (03) | The register and the store by name; no open shift; what waits to sync | `signed_out` with `REG01`, "الصندوق الاصطناعي 1", "متجر اصطناعي" / "Synthetic store" and `open_shift: null`; `sync.waiting: 1`, the store setup's own delivery row. |
| 4 | Lock with a shift open (03) | The lock screen names the shift's owner by name, never by staff number | `open_shift: { owner_display_name: "كاشير اصطناعي", business_date: "2026-11-16" }`; the cashier's staff number is not in the answer. |
| 5 | Fast user switch (03) | Another person signs in; the shift is still its opener's | The manager is signed in (`role: manager`), and `open_shift` is still the cashier's. |
| 6 | Waiting to sync (03) | The count grows with each fact graph | `sync.waiting` 1 before the shift opened, 6 after. |
| 7 | A wrong PIN (03) | The refusal the screen branches on | `auth_refused`, `message_key: error.auth.refused`, no detail. |
| 8 | The generated types hold (04) | Every answer type-checks; a broken one is refused | All seven answers compile as the generated type each command returns (`tsc` exit 0). The locked answer without its `register` is refused, naming `register` (exit 1). 18 generated files. |

## Not validated here, and why

- **The screen** (numpad, PIN pad, routing, Arabic and English) is part two; this part changes only what the register answers.
- **A real sync status** (online, offline, delivered) arrives with sync in Phase 3. Phase 1 only counts.
