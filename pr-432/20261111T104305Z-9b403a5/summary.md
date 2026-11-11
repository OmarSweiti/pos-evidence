# Validation: rustix 1.1.6, for RUSTSEC-2026-0486

**Commit validated:** `9b403a5`, the pull request's head. It was checked out clean in a scratch worktree, beside a clean checkout of `development`.

**What changes at runtime:** one transitive dependency's patch version.
- **The version:** `rustix` 1.1.4 → 1.1.6. The new advisory, RUSTSEC-2026-0486, is rated "unsound": the xattr functions mark unwritten bytes as initialised when given a zero-length buffer. It fails CI's required `supply-chain` check for every pull request.
- **The lockfile diff is rustix's version and checksum.** `cargo update -p rustix` also re-pointed eight unrelated dependency edges to other versions already in the lockfile; they were left out. `cargo metadata --locked` accepts the two-line change as it is.

**Where it runs:** the register reaches rustix only on Linux, through the keyring's secret-service store (`keyring` → `secret-service` → `zbus` → `async-io`/`polling` → `rustix`). Nothing the macOS register builds depends on it, so a local app run would not exercise it; `tempfile` reaches it in tests. The Linux build and its tests are CI's `rust` job. No code in this repository calls the functions the advisory names.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo deny check` on a clean checkout of `development` at `2f49a42` | the advisory refused, as the required `supply-chain` check reports | **refused**: `error[unsound]`, RUSTSEC-2026-0486, `advisories FAILED`, exit 1 (01) |
| 2 | `cargo deny check` on this branch at `9b403a5` | advisories, bans, licences and sources all ok | **pass**: exit 0 (01) |
| 3 | `git diff development -- Cargo.lock`; `cargo metadata --locked` | one version and its checksum; the lockfile consistent as it is | **as expected**; exit 0 (01) |
| 4 | `cargo tree -i rustix` on every target, on this Mac's, and on Linux from the register | reached through `zbus` on Linux and `tempfile` in tests; nothing on macOS | **as expected** (02) |
| 5 | CI's checks on this pull request | every required check green, `supply-chain` and the Ubuntu `rust` job included; `validation` waiting for this comment | **pass** (02) |
| 6 | `just pr`'s gate at this head | lint, test, build-web, guards and secrets pass | **pass**: 1037 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test (02) |

The two checksums are shown shortened on board 01; the full values are in the diff.

## Not validated, and why

- **The Linux secret-service store against a running keyring daemon.** Neither this Mac nor CI's runner provides one, and no test reaches it. The change is a patch-level step within one major version, and only the version and checksum changed.
