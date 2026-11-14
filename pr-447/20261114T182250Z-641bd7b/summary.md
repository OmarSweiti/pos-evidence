# Validation: #447, 1.8.6(b), restore with the recovery code alone

Validated commit: `641bd7b` (the pull request's head), from a clean scratch worktree.

## How it was run

- **Restore has no screen yet; 1.8.7's recovery screen will call it.** It was driven around the till that will use it. An evidence-only harness, linked against this head's `terminal` and `pos-db` and not part of the repository, sells through the till's own commands, backs up with part one's `backup_service::take`, and calls `pos_db::backup::restore`. Each harness verb is one process, and the till is opened with `Register::open_in`, as `run` opens it. This head's `verify-audit` binary checks the result.
- **No hardware is involved:** a receipt is queued, not printed, and no device emulator runs.
- **Machines:** scratch directories, started empty. The register is the shell suites' own shop (synthetic data, the suites' synthetic database key). The key sits in a file standing in for the OS credential store, so this Mac's real Keychain was never touched. A real recovery code wraps the key into the confirmed envelope. The code goes to a "paper" file and is never printed.
- **The till's clock is the shop's fixed instant** (5 November 2026, its seeded shift's day), so the generation is named for it. The restore's quarantine is named for the moment it ran.
- **Masking:** every key, digest, id and local path in the output is masked.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db backup::` (the step's `Done when`) once and three times more; part one's `-p terminal backup_service::`; `-p pos-db key_custody::`; every `pos-db` and `terminal` test; clippy; `just pr`'s gate | all pass | 12, 4 and 12 pass on every run; 544 pass; clippy exit 0; the gate is clean (1088 workspace tests) |
| 2 | The till sells two, takes its backup, sells a third, and is killed with `abort()` | a verified generation holding two sales; the third sale only in the `-wal` | REG01-000001 and -000002 sold, generation published and anchored, REG01-000003 sold after; `-wal` and `-shm` beside the database; exit 134 |
| 3 | The credential store loses the key | the till refuses to open, naming the recovery code | refused: "opened with the recovery code, never with a new key" |
| 4 | Restore with a code that is not the paper's, then from a copy of the generation cut short by a page | each refused by name; every file at the path keeps its bytes | "the recovery code does not open this envelope"; "its database is not the size its manifest records"; untouched both times |
| 5 | Restore the generation over the register's own files with the paper's code | the database, log and index set aside unchanged; nothing left beside the restored file; the chain intact to the anchor; the till opens on the backup's two sales | as expected; `Intact { entries: 3 }`; the till opens with 2 sales and their 26 delivery rows waiting |
| 6 | Open the files set aside, from a copy | they still hold the third sale: not lost, and not replayed into the restored database | 3 sales, 37 delivery rows waiting |
| 7 | Carry the generation to a new machine and restore it there with the paper's code alone | nothing set aside; every table row for row; the till opens | 63 tables, 313 rows, identical; the till opens with both sales |
| 8 | `verify-audit --anchor <generation>/backup.json` on a copy of the new machine's database | intact through the anchored head | INTACT, 3 entries, anchored through the head; exit 0 |
| 9 | 15 planted defects in a scratch worktree at this head's code | each caught by its named test | 15 of 15 caught, every mutant compiled |

## Not validated, and why

- **Writing the key, envelope and device id back** is 1.8.7's, with its recovery screen and command. The harness does it by hand where the till must open, and says so.
- **A restore stopped part-way** is shown by the unit test `a_restore_stopped_before_activation_finishes_when_run_again`, not as a drill: stopping between two file renames needs the test's window on that moment.
- **The off-machine destination** is 1.8.6b's.
