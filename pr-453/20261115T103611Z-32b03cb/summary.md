# Validation: #453, 1.8.8, the disk-space guard

Validated commit: `32b03cb` (the pull request's head), from a clean scratch worktree.

## How it was run

- **The guard has no screen yet; the status strip and the sale screens are 1.11's.** An evidence-only harness, linked against this head's `terminal` and not part of the repository, opens the suites' shop register with the operating system's own free-space reading (`SystemDisk`, through `fs4`), never a fixture, and drives `cart_open`, `tender_begin`, `sale_finalize`, `health_status` and the backup worker's `take` through the functions their handlers call. Each harness verb is one process.
- **The disk is real:** a 1 GiB HFS+ disk image attached with `hdiutil` at a folder in the scratch directory, holding the register's database. The drill fills it with a file of zeros until about 300 MiB are free, then removes the file. A second register on this Mac's own disk shows a disk with plenty of room.
- **The register** is the shell suites' own shop (synthetic data, the suites' synthetic database key). No key or path is printed; only the disk's part of `health_status` is shown, since the shop has no printer.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal low_disk_` (the `Done when`) once and three times more; every `terminal` test; clippy; `cargo deny` on the new crate; `just pr`'s gate | all pass; `fs4` passes licences and advisories, pulled in by the terminal only | 4 pass on every run; 186 pass; clippy exit 0; deny ok; `fs4` ← `terminal`; the gate is clean (1109 workspace tests) |
| 2 | A register on this Mac's disk | plenty; no disk alarm | `Plenty`, about 15 GiB free; no disk alarm |
| 3 | A register on the 1 GiB disk image | below 2 GiB: `disk_low`, trade goes on | `Low`, 998 MiB free; `disk_low` |
| 4 | A payment begun, then the disk filled to about 300 MiB free | `disk_full`, first of all alarms; the sale being paid for finishes; a new basket refused, nothing written | `Full`; `disk_full` first; REG01-000001 completed; `disk_full` refusal with why; every table unchanged |
| 5 | The hourly backup on the full disk | not taken; no folder made | refused, saying a copy would take a sale's space; no `backups/` |
| 6 | The filler removed | `low` again, warning only; a sale completes | `Low`; REG01-000002 completed |
| 7 | 12 planted defects in a scratch worktree at this head's code | each caught by its named test | 12 of 12 caught, every mutant compiled. A first run at the previous head caught 11: moving `disk_full` below the printer-binding alarm survived because the test's register had no binding. The test now gives it one, and the whole set was run again at this head. |

## Not validated, and why

- **The screens** that show the alarms and the refusal are 1.11's: the harness reads `health_status` and the commands' answers.
- **Windows and Linux readings** were not run here; the same `fs4` call reads them, and CI's promotion builds compile it.
- **An unreadable disk** is shown by the unit test `low_disk_unknown_space_never_blocks_a_sale`, not as a drill: no real volume refuses a free-space query on demand.
- **`protected-paths` is red by design.** `scripts/check-test-catalog.py` is in the frozen policy set, and this PR moves 1.8.8's held test name out of `PLANNED`; the CI log names that file alone.
