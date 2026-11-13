# Validation: #445, 1.8.6(a), verified encrypted backups taken hourly by the till

Validated commit: `ce4cdd5` (the pull request's head), from a clean scratch worktree.

## How it was run

- **The worker has no screen; it runs beside the till.** It was driven as `run` drives it. An evidence-only harness, linked against this head's `terminal` and `pos-db` and not part of the repository, opens the till with `Register::open_in` and starts `backup_service::start` as `run` does. It also calls `backup_service::take`, `pos_db::backup::{verify, snapshot, prune}`, and this head's `verify-audit` binary. Each harness verb is one process.
- **No hardware is involved:** no printer or other device emulator.
- **Machine:** a scratch directory, started empty. The register is the shell suites' own shop (synthetic data, the suites' synthetic database key). The key sits in a file standing in for the OS credential store, so this Mac's real Keychain was never touched. A real recovery code wraps the key into the confirmed envelope. The code goes to a "paper" file and is never printed.
- **Masking:** every key, digest, id and local path in the output is masked; generation names are instants.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db backup::` and `-p terminal backup_service::`, each once and three times more; `-p pos-db key_custody::`; every `pos-db` and `terminal` test; clippy; `just pr`'s gate | all pass | 8, 4 and 12 pass on every run; 540 pass; clippy exit 0; the gate is clean (1084 workspace tests) |
| 2 | The till opens, a cashier signs in, the worker starts | a generation within seconds: database, envelope, manifest; it verifies with the live key; its head anchored with its delivery envelope | 1 generation within 5 s; the three files; verified; 1 checkpoint row, 1 with its envelope |
| 3 | `verify-audit --database <copy> --anchor <generation>/backup.json` | the manifest reads as an anchor; the chain is intact through it | anchor from verified_backup; INTACT, anchored through the head; exit 0 |
| 4 | The paper's code alone opens the generation's envelope and database | opens without the credential store; another code is refused | opens, read-only, products and users present; another code refused |
| 5 | A bench copy cut short, a bit flipped, the manifest rewritten to match; another key | each refused, naming why; the published generation still verifies | four refusals, each named; the published one verifies |
| 6 | The till opens again: a backup with no new audit row, then one after a sign-in | the first anchors nothing new; the second anchors the new head | anchored false, then true; 2 checkpoint rows, each with its envelope |
| 7 | Generations at chosen instants, then `prune` | over 30 days removed; the older of two on a day, and of two in an hour within the day, removed | as expected; the newest of each kept |
| 8 | A seeded register with no confirmed envelope | no backup, no folder | refused by name; no `backups/` |
| 9 | 16 planted defects in a scratch worktree at this head | each caught by its named test | 16 of 16 caught, every mutant compiled |

## Not validated, and why

- **The step's `Done when`** (a restore that verifies every sale, commit manifest and audit anchor) is the second part's, by the owner's decision of 13 November 2026.
- **The off-machine destination and the health report** are 1.8.6b's.
- **The give-way under a real concurrent sale** is shown by the unit test `a_waiting_command_is_seen_by_the_backup_and_served_after_it` and by `snapshot_serializes_with_source_writes_and_is_consistent`'s give-way half, not as a drill: a drill depending on thread timing would not be reproducible.
- **`protected-paths` is red by design.** `scripts/check-test-catalog.py` is in the frozen policy set, and this PR moves 1.8.5b's held test name from `PLANNED` to its retired list. The CI log names that file alone.
