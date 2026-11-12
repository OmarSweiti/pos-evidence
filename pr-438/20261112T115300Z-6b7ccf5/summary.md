# Validation: #438, 1.8.5b(a), the recovery code and the key envelope it opens

Validated at `6b7ccf5`, the pull request's head, in a clean scratch worktree. This part is `pos-db` only:
it has no screen, and the screen is 1.8.5b's fourth part, after 1.11.3b. So the drill is driven by an
evidence-only harness linked against the worktree's own `pos-db`, one process per step, on synthetic
files in a scratch directory. The harness is not part of the repository.

**The machine's real Keychain is never touched.** The OS credential store is stood in for by a file
beside the machine's data, through the `CredentialStore` seam this part adds. The recovery code is
written to a "paper" file and never printed. The key, the salt, the key's id and the wrapped key are
never printed either; the envelope is shown with those three values given by size.

## Commands

- `cargo nextest run -p pos-db key_custody::`: once in full and three times more.
- `PROPTEST_CASES=65536 cargo nextest run -p pos-db 'key_custody::prop_'`
- `cargo nextest run --release -p pos-db release_build_ignores_env_key_and_falls_through`
- `cargo nextest run --no-fail-fast -p pos-db -p terminal`, and clippy over both with `-D warnings`.
- `ev-key-harness mint | issue | wipe-store | open | lose | other-code | restore | envelope`
- `just pr`'s gate (`just pre-push`) at `6b7ccf5`, quoted from its log.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | The `key_custody::` filter | 11 pass, among them the step's fixture backup | 11 of 11, and again in each of 3 more runs (board 01) |
| 2 | The three properties at 65,536 cases | Pass | 3 of 3 (board 01) |
| 3 | 1.8.5's release-build test | Passes in a release build | Passed (board 01) |
| 4 | Every `pos-db` and `terminal` test; clippy | All pass; clean | 519 passed; clippy exit 0 (board 01) |
| 5 | `just pr`'s gate | Lint, test, build-web, guards, secrets | 1056 passed, 4 skipped; catalog and frontier reconcile (board 01) |
| 6 | A clean machine mints its key | Minted and kept | `CredentialStore`: minted and kept (board 02) |
| 7 | A code is issued and the key wrapped | The code goes to paper only; the code and key print as redacted | `#####-…` on the board; `RecoveryCode(redacted) · DatabaseKey(redacted)` (board 02) |
| 8 | The credential store loses its key, beside its database | Opening is refused by name, and nothing is minted | `KeyMissingForExistingDatabase`'s message; the store still holds no key (board 02) |
| 9 | A valid code that is not this envelope's | Refused | "the recovery code does not open this envelope" (board 03) |
| 10 | The right code with one digit of group 3 mistyped | Refused, naming the group and no digit | "group 3 does not check" (board 03) |
| 11 | The right code typed in Arabic-Indic digits with spaces, after the machine is lost | The envelope opens and the copy opens with its rows | Opened; all three rows (board 03) |
| 12 | Ten planted defects | Each turns its named test red, then is restored | 10 of 10 caught, every mutant compiled, the worktree restored to 0 changed files (board 04) |

## Not validated, and why

- **The OS credential store itself.** The drill uses a file in its place, deliberately: a test or drill
  that writes a key into this machine's Keychain is worse than the gap it closes. `OsCredentialStore`
  is the `keyring` call it was before, behind the new seam.
- **A screen, a printed sheet, and first-run provisioning.** These are 1.8.5b's parts (b), (c) and
  (d).
- **`protected-paths` is red by design.** `scripts/check-test-catalog.py` is byte-pinned, and two
  landed test names leave its `PLANNED` list.
