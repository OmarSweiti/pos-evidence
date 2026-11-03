# Validation: 1.10.3, the stock cache held to its ledger

**Commit validated:** `3008a5e`, the pull request's head, checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **`StockRepository::verify_cache(tx, store)`** compares every column of every `stock_cache` row with a full replay of the ledger, and names each product that differs.
- **`StockRepository::reconcile_cache(tx, store)`** verifies and, on any drift, rebuilds in the same transaction.
- **`rebuild_cache`** now shares its replay query with the check.
- **`pos_domain::on_hand`** is the sum of a product's deltas.

No screen or command calls these yet: start-up's call is 1.8.4's. The runtime is therefore shown through the repository itself, on synthetic register files.

**How it ran:**
- **The suites (01).**
  - The `Done when` command, `cargo nextest run -p pos-db --test stock`.
  - The domain's `on_hand` tests.
  - Both properties again at `PROPTEST_CASES=20000`.
  - 1.10.2's stock suite, every `pos-db` test, clippy, and the whole workspace.
- **The harness (02, 03, 04).**
  - It is linked against this head's `pos-db` and `pos-domain` and built `--offline` from the worktree's `Cargo.lock`.
  - Every step is its own process, run one after another on one register file, with each process's exit status recorded beside its output. A step that dies calls `abort()`, so no destructor runs and nothing commits.
  - The harness enables rusqlite's `hooks` feature, which adds no crate, only to kill start-up inside the rebuild: an update hook calls `abort()` at the rebuild's first insert. The shipped build doesn't enable it.
- **The planted defects (05).** 17 defects, each applied to a scratch worktree at `605420f`, tested, and restored. The only commit after `605420f` is `3008a5e`, which changes one sentence in the phase file. No migration was mutated.

Data is synthetic throughout: an invented store, two registers (`REG01`, `REG02`) and four products (`SYN-OIL`, `SYN-RICE`, `SYN-SUGAR`, `SYN-FLOUR`).

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db --test stock` | 5 pass | **pass**: 5 of 5 (01) |
| 2 | The domain's `on_hand` tests; both properties at 20,000 cases; 1.10.2's stock suite; `pos-db`; clippy; the workspace | all pass | **pass**: 3 of 3; 2 of 2 at 20,000 cases; 8 of 8; 294 of 294; clippy clean; 833 passed, 4 skipped (the existing `#[ignore]` ones). In this run nextest flagged one existing test, `auth::tests::a_new_pin_has_the_digits_its_account_needs`, as leaky. It passed, and this change doesn't touch it; the first capture at the same head had no leak (01) |
| 3 | Eight stock events in four commits, then the check | the cache is the replay | **pass**: on-hand 29.000, 9.750 and 9.000 as the ledger sums; `verify_cache: []` (02) |
| 4 | The cache written to by hand, four ways: an on-hand raised; a row one event behind; a row deleted; a row for a product no event moved | the till reads the wrong figures, and each is found by name | **pass**: the till reads 34.000, 0.000, 10.000 and 7.000. `verify_cache` returns all four: `Differs(SYN-OIL: on_hand_milli)`, `Missing(SYN-RICE)`, `Differs(SYN-SUGAR: on_hand_milli, last_event_seq, last_event_id, event_count)` and `Unexplained(SYN-FLOUR)` (02) |
| 5 | Start-up dies inside the rebuild, after its delete, while inserting the first replayed row | the next process finds the old cache whole | **pass**: exit 134 after 3 rows were deleted. The next process finds the same three rows and the same four drifts (03) |
| 6 | Start-up rebuilds, then dies before COMMIT | the next process finds the old cache whole | **pass**: alarm raised, 6 row changes made, exit 134. The next process finds the same four drifts (03) |
| 7 | Start-up reconciles and commits; then starts again | an alarm naming the four drifts, then a true cache; the second start writes nothing | **pass**: the alarm names all four and the rebuild commits. The cache is back at 29.000, 9.750 and 9.000 with no flour row, and `verify_cache: []`. The second start reports `[] (true); row changes 0` (03) |
| 8 | Another register's event recorded in this register's file, then start-up | refused by name, the cache left as it was | **pass**: `reconcile_cache` and `verify_cache` are both refused as `StockReplayAcrossRegisters`. The cache is as the trigger projected it (04) |
| 9 | 17 planted defects, one predicted before the run to survive | each named defect turns its test red | **pass**: 16 of 16 caught, and M15 survived as predicted. Every mutant compiled, and the worktree was restored with no changed file (05). In the first run, M01 and M02 did not compile because of a dereference error in my replacement code. I fixed the replacement and re-ran those two alone, and both were caught |

`just pre-push` was run by `just pr` before the pull request was opened, and passed: lint, tests, the web build, guards and the secret scan.

## Not validated, and why

- **A register calling the check at start-up.** `reconcile_cache`'s caller is 1.8.4's start-up orchestration, which doesn't exist yet. A note at 1.8.4 says how it calls this. The harness's `start` step makes the call the way that note describes.
- **A periodic sweep.** I-6 asks for one, and no microstep schedules it yet. The pull request says so.
- **`sale` events in the property.** The property excludes them. The deterministic `Done when` test replays real completed sales instead, including a scale label's derived weight and stock sold below zero (step 1).
