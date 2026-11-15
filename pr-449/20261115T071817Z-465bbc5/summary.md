# Validation: #449, 1.8.6b, a verified copy of every backup off the machine

Validated commit: `465bbc5` (the pull request's head), from a clean scratch worktree.

## How it was run

- **The backup worker has no screen; it runs beside the till.** An evidence-only harness, linked against this head's `terminal` and `pos-db` and not part of the repository, opens the till with `Register::open_in` and starts `backup_service::start` as `run` does. It also calls `backup_service::take` for later runs, `terminal_lib::health::health` (the `health_status` handler), and `pos_db::backup::restore`. This head's `verify-audit` binary checks the result. Each harness verb is one process.
- **The drive is a real second file system:** a 32 MB disk image created and attached with `hdiutil` at a folder in the scratch directory, unplugged with `hdiutil detach` and plugged back in with `hdiutil attach`. Its device differs from the register's disk. No other hardware or emulator is involved.
- **The register** is the shell suites' own shop (synthetic data, the suites' synthetic database key). The key sits in a file standing in for the OS credential store, so this Mac's real Keychain was never touched. A real recovery code wraps the key into the confirmed envelope. The code goes to a "paper" file and is never printed.
- **The setting is written from outside the till,** as no command sets it yet (`settings_set` has no owning step).
- **Masking:** every key, digest, id and local path in the output is masked. Only the backup half of `health_status` is shown; the shop has no printer, so the printer's own alarms are left out.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db backup::` (the `Done when`), `-p terminal backup_service:: health::tests` and `-p pos-db --test setting`, each once and three times more; every `pos-db` and `terminal` test; clippy; `just pr`'s gate | all pass | 15, 9 and 6 pass on every run; 553 pass; clippy exit 0; the gate is clean (1097 workspace tests) |
| 2 | Before the worker's first run, read health | both destinations "not yet read", no backup alarm | as expected |
| 3 | The worker starts | within seconds, one generation in both places; the drive's copy is the local one byte for byte and verifies there with the live key; both current; no backup alarm | first run within 5 s; same bytes, verified; both `current`; alarms `[]` |
| 4 | The drive unplugged; the next run | the local backup is still taken; the drive's copy fails with why; its folder is not created; `backup_off_machine_missing` alone | "the backup destination is not there, and is never created"; folder not created; `unavailable`; that alarm alone |
| 5 | The setting pointed at a folder on the register's own disk | the copy is made, but the folder reads as on the register's disk and the alarm stays | `on_register_disk`; the alarm stays |
| 6 | A setting of the wrong shape, then none | nothing copied; `setting_unreadable`, then `not_set_up`; the alarm stays | as expected |
| 7 | The drive plugged back in, the setting restored | the copy reaches it, retention applies there, and the alarm clears | copied, the older generation pruned on the drive; both `current`; alarms `[]` |
| 8 | Machine A's local backups, database and key deleted; restore on machine B from the drive with the paper's code | restores; audit intact; every table row for row; the till opens | `Intact`; 63 tables, 251 rows, identical; the till opens |
| 9 | `verify-audit --anchor <the drive's generation>/backup.json` on a copy of B's database | intact through the anchored head | INTACT, anchored through the head; exit 0 |
| 10 | 20 planted defects in a scratch worktree at this head's code | each caught by its named test | 20 of 20 caught, every mutant compiled |

## Not validated, and why

- **Staleness after 26 hours** is shown by the unit tests (`an_unplugged_drive_is_reported_and_the_local_backup_still_taken` moves the till's clock 26 hours; `backup_alarms_are_raised_for_each_copy` covers every combination), not as a drill: it would need a day of real time or a clock the drill moves.
- **A network path** was not exercised; it is a folder like the drive's to the code, and its reachability is read the same way.
- **Windows' same-disk check** (by the path's root) is not exercised on this Mac; CI's promotion builds compile it.
- **Writing the key, envelope and device id back after a restore** is 1.8.7's; the harness does it by hand where the till must open, and says so.
