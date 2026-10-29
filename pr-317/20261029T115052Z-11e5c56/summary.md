# Validation: microstep 1.4.5, the caps in the shipped role grid

**Commit validated:** `11e5c56`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- **Real databases:** every database is a real SQLCipher register database, opened by the pull request's own `pos_db::open` with a synthetic key in a scratch directory, and deleted afterwards.
- **Upgrade fixtures:** the "selling register" databases are built at v7 from the committed migrations `0001`–`0007` and then opened, so `0008` runs over the cells `0004` shipped. No device or emulator is involved.
- **A throwaway harness**, not in the repository, called `RoleRepository::discount_cap` and handed each cap to #315's `apply_line_discount`.
- **Planted defects:** each one changed `crates/pos-db/src/repo/role.rs` or the domain grid in `permissions.rs`, ran the migration and role-matrix tests, and was restored. The run ends with "restored: 0 changed files", and the harness was built after it.
- **The committed migration was never mutated,** even in the scratch worktree. The repository's immutability guard protects it, and board 02 shows its behaviour directly.
- **Postgres:** the engine pass did not run on this machine, because Docker was not running. Board 01's last block is copied from CI's `rust` job log for `11e5c56`, where it ran against a real PostgreSQL service.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db --test migration_0008_discount_caps --test role_matrix` (the `Done when`'s database half), then its line half from #315 | 5/5, then 36/36 | **pass** (01) |
| 2 | `pos-domain`, `pos-db`, the workspace | no failure | **pass**: 292/292, 239/239, 691/691 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | `verify-schema.py --verbose` | `0001`–`0008` applied from disk; `§0008` skipped as shipped; the planned blocks `0009`–`0013` apply after it | **pass** (01). It lists `0001`–`0008` twice because it applies the chain in two passes |
| 4 | The Postgres mirror: mapping here, engine pass in CI | 8 mirrors for 8 migrations; all 8 applied to real PostgreSQL | **pass** (01; the engine pass is CI's) |
| 5 | A new register database | `user_version` 8; cashier `max_percent_ppm` 50000, shift lead 150000, manager uncapped, owner `sets_the_limit`; read back as 5%, 15%, no cap, not held | **pass** (02) |
| 6 | A database at v7 as `0004` shipped it, then opened; `0008`'s statements replayed | the same cells as step 5; the replay changes 0 rows | **pass** (02) |
| 7 | A database at v7 whose merchant set a 2% cashier cap and withheld the shift lead's discount, then opened | both edits stand, and are read back as 2% and not held | **pass** (02) |
| 8 | Eight cells the reader cannot trust: no number, text, a fraction, −1, above 100%, another kind with a number, a number with no kind, not JSON | each refused by name, none read as a cap or as unlimited | **pass** (02) |
| 9 | Each role's cap from the database, handed to `apply_line_discount` on a line worth 2.500 with no approval, at 5%, 5.0001%, 15%, 15.0001% and 50% | cashier only 5%; shift lead up to 15%; manager all; owner no discount at all | **pass** (03) |
| 10 | The renumbered plan | `schema.md`'s table and headings run `0002`–`0013` with `0008` the discount caps; no old planned file name left in any tracked file outside the immutable source plans | **pass** (03): 0 references to each old name |
| 11 | Eleven planted defects in the reader and the domain grid | each caught | **pass** (04): all eleven caught, worktree restored |

## Notes

- **A first run at `6c9651e` let one planted defect survive:** disabling the reader's check of the limit's kind. The test's `{"kind":"own_shift"}` carried no number, so it was refused for that reason instead. `11e5c56` tests a cell of another kind with a number beside it, and a number with no kind. All eleven are caught at this head, and every result here is from `11e5c56`.

## Not validated, and why

- **Combining a user's several roles into one cap** (1.8.x), and **the `cart_discount_line` command** (1.4.11), which reads the cap with this repository: neither exists yet.
- **The Postgres engine pass on this machine:** Docker was not running. CI ran it.
