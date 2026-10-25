**Validated at `b8716d7`.** Microstep 1.2.7: the search benchmark. It also brings the product repository's cached save.

**Setup**
- **Code under test:** a clean scratch worktree of this exact head.
- **Harness:** throwaway, outside the repository, linking this head's `pos-db` with its `fixtures` feature. It uses the repository's own `Cargo.lock`.
- **Database:** a fresh encrypted register database opened through `pos_db::open`, loaded with this head's `search_catalogue::seed`. That is the same catalogue the benchmark and its tests use.
- **Synthetic data only:** generic grocery words and invented brands.
- **Timings:** an Apple M3 laptop (16 GB, macOS 26.5.2), release build. Per conventions §7.2 they are indicative and never a baseline.
- **No app or emulator:** search is a repository function until the `catalog_search` IPC command is wired, so there is no screen to drive.

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db search_benchmark_` (Done when, first half) | 2/2 pass | ✅ Board 01 |
| 2 | `cargo test -p pos-db --bench search` (Done when, second half) | each of the four lengths runs once and succeeds | ✅ Board 01 |
| 3 | Every `pos-db` test, which includes the cached save; then the workspace | all pass | ✅ Board 01: 209/209 and 573/573; the 4 skipped are the repository's four `#[ignore]` tests (device and child-process targets), and this PR adds none |
| 4 | The catalogue in the database | 50 000 live, distinct and indexed; 5 000 with tashkeel; 35 300 the fold changes; the file is ciphertext | ✅ Board 02 |
| 5 | The four queries `ا`, `ال`, `الو`, `الواد` | 1, 2, 3 and 5 characters; 50 000, 50 000, 4 000 and 2 000 matches; a full page of 20 each | ✅ Board 03 |
| 6 | `cargo bench -p pos-db --bench search` | a time for each length | ✅ Board 04: 65.4, 63.1, 7.4 and 1.8 ms. **The 1- and 2-character figures are above the 50 ms limit.** |
| 7 | Where the time goes: an evidence-only probe, swapped in and restored | matching is cheap; ranking every match is not | ✅ Board 04: 1 char, find every match 5.1 ms; first page unranked 3.1 ms; rank inside the index 39.3 ms; the repository's whole query 58.5 ms |
| 8 | Five one-line defects in the fixture, the queries and the search | each caught | ✅ Board 05 |
| 9 | The cached save undone | correct, so no test fails; the load is slower | ✅ Board 05: both tests pass, and each takes about 34 s instead of about 6 s |

Step 6 is the finding the PR leads with. It is a development-laptop figure, so it proves nothing about the reference register, but it is above the risk-register trigger in `00-master-plan.md` §6: a quarter of the limit, 12.5 ms. The response is the owner's decision.

CI at this head: `rust`, `printer-conformance`, `guards`, `web`, `supply-chain`, `protected-paths`, `topology`, `label`, and CodeQL with its five analyses are green, with **0 open code-scanning alerts**.

**Not validated here, and why**
- **Any figure on the reference register, or `just bench-gate search`.** There is no reference register before milestone 5.8; 5.8.2 records the baseline and proves the limit.
- **The `catalog_search` IPC command and a screen.** Not wired yet.
- **A fix for the slow first keystrokes.** None is in this PR, by design; that decision is the owner's.
