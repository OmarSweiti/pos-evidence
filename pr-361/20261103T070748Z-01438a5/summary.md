# Validation: move off the yanked async-executor 1.14.0

**Commit validated:** `01438a5`, the pull request's head, checked out clean in a scratch worktree.

**What changes at runtime:** one dependency's version. `async-executor` goes from 1.14.0, which was yanked on 2 November, to 1.13.3, which `zbus` 5.19.0's `^1.11.0` accepts and which has the same dependency list. The lock diff is the version and its checksum.

**Where it runs:** only on Linux. It reaches the register through `keyring`'s Linux secret-service store (`keyring` → `zbus-secret-service-keyring-store` → `secret-service` → `zbus`). On this Mac's target nothing depends on it, so a local app run would not exercise it. The Linux build is CI's.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo deny check` on a clean checkout of `development` at `687bcf1` | the yank refused, as the required `supply-chain` check reports | **refused**: `error[yanked]`, `advisories FAILED`, exit 1 (01) |
| 2 | `cargo deny check` on this branch at `01438a5` | advisories, bans, licences and sources all ok | **pass**: exit 0 (01) |
| 3 | `git diff development -- Cargo.lock` | one version and its checksum | **as expected** (01) |
| 4 | `cargo tree -i async-executor`, on every target and on this Mac's | reached only through `keyring`'s Linux store; nothing on macOS | **as expected** (02) |
| 5 | CI's `rust` job on this pull request, on its Ubuntu 26.04 runner | 1.13.3 downloaded, checked under clippy `-D warnings`, compiled, and the workspace's tests pass | **pass**: 800 passed, 4 skipped (the existing `#[ignore]` ones) (02) |
| 6 | Every required check on this pull request | green, `supply-chain` included | **pass**: rust, guards, web, supply-chain, printer-conformance, protected-paths and topology |

`just pre-push` was run by `just pr` before the pull request was opened, and passed.

## Not validated, and why

- **The Linux secret-service store against a running keyring daemon.** Neither this Mac nor CI's runner provides one, and no test reaches it. The change is a patch-level step back within one major version, with an unchanged dependency list.
- **1.14.1.** It was published on 2 November, inside the seven-day dependency cooldown. Dependabot can propose it once it clears.
