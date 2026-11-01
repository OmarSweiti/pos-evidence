# Validation: microstep 1.8.0, the storage-engine gate

**Commit validated:** `0214087`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- **A storage-layer change** with no screen. The gate runs inside `pos_db::open`, so the evidence is a real register database opened by the pull request's own code, plus the gate's decision put to engines no build here produces.
- **A throwaway harness**, not in the repository, built offline against the repository's own `Cargo.lock`, which pins `rusqlite` `0.39.0`. It:
  - opened a temporary register database with a synthetic key;
  - read the passage of `ref/plan-validation.md` §5 from the pull request's own copy and hashed it again;
  - put fifteen engines to `storage_verdict`;
  - swept every version around both floors against the floors stated separately as tuples.
- **Planted defects:** each one changed one place in `crates/pos-db/src/storage_version.rs` or `crates/pos-db/src/lib.rs` in the worktree, ran every `storage_version::` test, and was restored. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db storage_version::` (this step's `Done when`) | the six tests pass | **pass**: 6/6 (01) |
| 2 | The `pos-db` crate, the workspace | no failure, every open now passing through the gate | **pass**: 245/245; 761/761 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | A register database opened by `pos_db::open` | the engine read as SQLCipher `4.10.0 community` and SQLite `3.50.4`, single-connection mode, WAL on | **pass** (02) |
| 4 | The policy's source | the passage of §5 hashed again matches the pinned digest; the same passage with one version changed does not; the passage names all four minimums | **pass** (02) |
| 5 | Fifteen engines put to the decision | older than the reviewed build, or unreadable, refused by name; reviewed or newer without the fix, one connection; both with the fix, a second | **pass** (02) |
| 6 | Every version 2.0.0 to 5.60.12 of each engine, the other at its reviewed build or at the fix; every three-number version to 20.20.20 parsed | 0 differing from the floors stated separately; every version read as itself, its two- and four-number forms refused | **pass**: 12,688 cases, 0 differing; 9,261 parsed, 0 wrong (02) |
| 7 | Fourteen planted defects | each caught | **pass** (03): all fourteen caught, worktree restored |

## Notes

- **The design question the entry left to its builder is answered with two tiers.** The floor to open at all is the reviewed build. The floor for a second source connection is the WAL-reset fix, so today's build opens in single-connection mode. The pull request explains why under "Read before approving".
- **The refusal happens before WAL.** The tests put each refused engine to the open itself and then find no `-wal` file and the journal mode unchanged. The planted defect that asks the engine only after WAL is on is caught by all three refusal tests.
- **Nothing opens a second connection today**, so "WAL is disabled for multiple connections" is shown as the gate's answer, not as a refused second open: no second open exists to refuse.

## Not validated, and why

- **The engine with the fix:** no build here carries it until #244. The second-connection tier is shown only through injected versions.
- **The diagnostics screen showing these versions** (1.11.9).
