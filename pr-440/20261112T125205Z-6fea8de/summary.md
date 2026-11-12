# Validation: #440, 1.8.5b(b), first-run custody

Validated at `6fea8de`, the pull request's head, in a clean scratch worktree. There is no screen yet:
the provisioning screen is 1.8.5b's fourth part, after 1.11.3b. So the till is driven by an
evidence-only harness linked against the worktree's own `terminal` and `pos-db`, one process per
step. Each step starts the till with `Till::open_in`, as `run` does, and calls what the two commands
call. The harness is not part of the repository.

**This Mac's real Keychain is never touched.** The OS credential store is stood in for by a file
beside the machine's data directory, through the `CredentialStore` seam. Each shown code goes to a
"paper" file and is never printed. The id it is shown under is masked on the boards.

## Commands

- `cargo nextest run -p terminal recovery_code_provisioning_`: once in full and three times more.
- `cargo nextest run -p pos-db key_custody::`
- `cargo nextest run --no-fail-fast -p pos-db -p terminal`, and clippy over both with `-D warnings`.
- `ev-firstrun-harness start | new | again | opens | lose-pending`
- `just pr`'s gate (`just pre-push`) at `6fea8de`, quoted from its log.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | The `recovery_code_provisioning_` filter | 5 pass, the step's three terminal tests among them | 5 of 5, and again in each of 3 more runs (board 01) |
| 2 | `key_custody::`, every `pos-db` and `terminal` test, clippy | All pass; clean | 12; 525; clippy exit 0 (board 01) |
| 3 | `just pr`'s gate | Lint, test, build-web, guards, secrets | 1062 passed, 4 skipped; the frontier reconciles 125 (board 01) |
| 4 | A clean machine, started twice | `not_set_up` / `new_machine`, nothing on disk, no key, register commands refused by name | As expected, both times (board 02) |
| 5 | "New register" | A key kept, the database made, the envelope pending, the code shown with its `Debug` redacted; asking again refused | As expected; the pending envelope opens with the paper's code to the kept key (board 02) |
| 6 | A restart before confirmation | `code_not_confirmed`; a fresh code; another id refused; its own id confirms | As expected; the first paper's code opens nothing, the second's opens the register's envelope (board 03) |
| 7 | A restart after confirmation | `store_not_set_up`; nothing issued again | As expected (board 03) |
| 8 | A first run whose pending envelope never reached the disk after its database did | `code_not_confirmed`, never a closed till | As expected (board 03) |
| 9 | Nine planted defects | Each turns its named test red, then is restored | 9 of 9 caught, every mutant compiled, the worktree restored to 0 changed files (board 04) |

## Not validated, and why

- **The OS credential store itself.** A file stands in for it, deliberately: a drill that writes a key
  into this machine's Keychain is worse than the gap it closes. The production path is
  `OsCredentialStore` through the same seam.
- **The Tauri command wrappers' IPC round trip.** The commands are one-line wrappers over the
  functions the harness calls, and the IPC contract test checks their registered arguments.
- **A screen and the printed sheet.** These are 1.8.5b's parts (d) and (c). The register's identity is
  1.8.5c's.
