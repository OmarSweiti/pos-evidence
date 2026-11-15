# Validation: #451, 1.8.7, keychain-loss recovery

Validated commit: `7785244` (the pull request's head), from a clean scratch worktree.

## How it was run

- **The recovery screen is 1.11.9b's; this step owns its state and commands.** An evidence-only harness, linked against this head's `terminal` and `pos-db` and not part of the repository, opens every till with `Till::open_in`, as `run` opens one, and drives `recovery_state` and `recovery_restore_backup` through the functions their handlers call (`backup_recovery::state`, `backup_recovery::restore`), plus `session_state`'s `state_of`. No one is signed in. Each harness verb is one process.
- **The drive is a real second file system:** a 32 MB disk image attached with `hdiutil` at a folder in the scratch directory, and detached at the end. No other hardware or emulator is involved.
- **The register** is the shell suites' own shop (synthetic data, the suites' synthetic database key). It sells through the till's own commands and is backed up by the till's own worker. The key sits in a file standing in for the OS credential store, so this Mac's real Keychain was never touched; "losing the key" deletes that file. A real recovery code wraps the key; it goes to a "paper" file and is never printed.
- **The till's clock is the shop's fixed instant** (5 November 2026), so the generation is named for it.
- **Masking:** every key, digest, id and local path in the output is masked; the recovery actor is described, not printed.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal recovery_` (the `Done when`) and `-p pos-db backup::`, each once and three times more; every `pos-db` and `terminal` test; clippy; `just pr`'s gate | all pass | 14 and 17 pass on every run; 561 pass; clippy exit 0; the gate is clean (1105 workspace tests) |
| 2 | A register sells two, unsynced; its worker backs up locally and to the drive | the drive's folder remembered beside the database | `backup-off-machine.json` names it |
| 3 | The key is lost; the till starts | `session_state` says recovery (`key_lost`); register commands refused as before; both copies listed without opening the database | as expected; local and remembered candidates, the remembered one on the drive |
| 4 | A wrong code; the paper's with one digit mistyped; the right code on a machine whose device id names another register | each refused by name; nothing touched, nothing set aside, no key written | `recovery_code_wrong`; `recovery_code_mistyped` (group 3); `recovery_backup_is_another_register`; untouched |
| 5 | The drive's generation restored with the paper's code | old files set aside; the key back in the store; `restored: true`; a second restore refused | as expected; `recovery_already_restored` |
| 6 | The till starts again | it opens signed out; the cashier signs in; both sales present, unsynced; the restore audited as the recovery actor | as expected; `backup.recovery_restore` by the recovery actor, payload `why: key_lost`, `set_aside: true`, chain `intact` |
| 7 | A clean machine chooses to restore and points at the drive | the drive's generation listed; restored; the envelope and device id put beside it; first run closed | as expected; the device id `device-REG01` from the backup's own row; a new code refused |
| 8 | The clean machine's till starts again | opens with both sales; the restore audited, `why: new_machine` | as expected |
| 9 | A wrong key, a damaged schema, a newer build's database | the first two wait for a restore; the third stays closed | `wrong_key`, `schema_damaged`; closed, `recovery_state` refused |
| 10 | 21 planted defects in a scratch worktree at this head's code | each caught by its named test | 21 of 21 caught, every mutant compiled |

## Not validated, and why

- **The screen** is 1.11.9b's: no UI exists yet to drive. The harness calls the commands' own functions.
- **Relaunching the app after a restore** is the screen's: the drill starts a new process instead, as a relaunch does.
- **A network folder** was not exercised; to the code it is a folder like the drive's.
- **`protected-paths` is red by design.** `scripts/check-test-catalog.py` is in the frozen policy set, and this PR moves 1.8.7's held test name out of `PLANNED`; the CI log names that file alone.
