# Validation: microstep 1.6.2, the pos-db half — staff PINs

**Commit validated:** `71dacf4`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- **A storage-layer change with no screen.** Signing in is the terminal half's. The evidence is the pull request's own `pos_db::auth` and `UserRepository`, on a register database opened by its own `pos_db::open`, with the fixture's cashier and two registers.
- **A throwaway harness**, not in the repository, built offline in release against the repository's own `Cargo.lock`. Every PIN, id and moment is synthetic. A PIN hash is on the never-log list, so even for a synthetic PIN the evidence shows only its parameters, with its salt and tag withheld.
- **Planted defects:** each one changed one place in `crates/pos-db/src/auth.rs` or `crates/pos-db/src/repo/user.rs` in the worktree, ran every `auth::` test, and was restored. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db auth:: && cargo test -p pos-db --bench pin_verify` (the `Done when`'s `pos-db` half) | every test passes, and the benchmark runs once | **pass**: 11/11; both benchmark cases run (01) |
| 2 | The `pos-db` crate, the workspace | no failure | **pass**: 260/260; 776/776 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | One timed benchmark run on a development laptop | a figure, indicative only | about 74 ms for the right PIN and the wrong one alike, criterion's estimate; no baseline (01) |
| 4 | A synthetic PIN hashed twice | Argon2id at `m=65536,t=3,p=4`; the two hashes and their salts differ; neither contains the PIN; the parameters read back equal the reviewed profile | **pass** (02) |
| 5 | PINs checked against that hash, plus a value that is no hash and a PBKDF2 hash | only the right PIN verifies; the other two refused by name | **pass** (02) |
| 6 | Hashes below the profile in each one parameter, and the common 19 MiB profile | each verifies, and each is flagged for a rehash; the reviewed profile is not | **pass** (02) |
| 7 | The digits a new PIN needs, and new PINs checked | four for a cashier, six for any refund, settings, user-admin or Z-report holder; anything short, non-digit or over twelve refused | **pass** (02) |
| 8 | The attempt policy, the owner's answer to merchant decision 9.7 | three free; then 30 s doubling to 900 s; locked from the tenth | **pass** (02) |
| 9 | Wrong PINs on one register, with the register restarted three times | the wait and the lockout survive each restart; the other register's count is untouched; a release clears the count, and that survives a restart too | **pass** (02) |
| 10 | The cashier's PIN replaced, then the account made inactive | the new PIN verifies and the old one doesn't; an inactive account has no PIN to check | **pass** (02) |
| 11 | Every count of wrong PINs 0–64 at five moments around its wait | 0 differing from the policy stated separately | **pass**: 325 cases, 0 differing (02) |
| 12 | Twenty-one planted defects | each caught | **pass** (03): all twenty-one caught, worktree restored |

## Notes

- **The planted defects found a gap, closed before the pull request opened.** The rehash test's weak hash was below three floors at once, so dropping any single check passed. `a_hash_below_the_profile_in_any_one_parameter_is_rehashed` now tests each floor alone, and catches each of the five defects.
- **The right PIN and a wrong one take the same time** to check, about 74 ms each here. A faster refusal would tell a guesser it was wrong before the hash was done.
- **The 74 ms is not the band.** Conventions §7 targets 200–350 ms median on the reference register, which doesn't exist before milestone 5.8. 5.8.2 measures there and re-tunes the profile if it falls outside the band.

## Not validated, and why

- **`user_reset_pin`, the release command and its audit row, and the login commands:** the terminal half's, waiting for the database to be wired into the terminal.
- **#233's second factor:** an open merchant question, whose default stands.
