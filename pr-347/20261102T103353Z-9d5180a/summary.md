# Validation: the js-minor group, with the @types/node override moved

**Commit validated:** `9d5180a`, the pull request's head, checked out clean in a scratch worktree. That's Dependabot's commit, `2ae8cf5`, plus one commit on top of it.

**What the extra commit does.**
- Dependabot raised `@types/node` to 24.19.2 in both apps' `package.json`. It can't edit `pnpm-workspace.yaml`, whose override still forced 24.13.3, so the pull request declared one version and installed the other.
- `scripts/check-node-version.py` (the `web` job) and the branch-workflow policy (`guards`, and `protected-paths` from the trusted revision) both refused it. Every step those jobs never reached was run on `2ae8cf5` and passed.
- The commit moves the override to 24.19.2, which is the typings release for the Node 24.19.0 runtime, and regenerates the lockfile. The lockfile's only other changes are `@types/node`'s integrity hash and the `undici-types` range it pins (06).

**How it ran:**
- Installed with `pnpm install --frozen-lockfile`, built with `pnpm -r build`, served on 127.0.0.1 only, on ports 4317 and 4318.
- Driven in headless Chrome, in a throwaway profile, with real mouse clicks and typed text.
- Every console error and uncaught exception was recorded: there were none.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | The terminal on load | the smoke panel, Arabic by default, right to left | **pass**: `lang=ar dir=rtl`, the columns from the right (01) |
| 2 | The language toggle clicked | English, left to right, and the toggle now offers العربية | **pass** (02) |
| 3 | A total of 2166 minor units, split into 3 | both accepted, no refusal | **pass** (03) |
| 4 | A total of `1.5` | refused by the money guard, with the last good value still the one to send | **pass**: "money must be whole minor units. Sending 2.166 JOD in 3." (04) |
| 5 | The back office on load | its placeholder page | **pass** (05) |
| 6 | Installed versions; `tauri info` | the five bumped packages at their new versions; the JS Tauri packages level with the Rust crate | **pass**: `@tauri-apps/api` and `cli` 2.12.4 beside `tauri` 2.12.5 (06) |
| 7 | The two checks that refused `2ae8cf5` | both pass | **pass** (06) |
| 8 | biome, the JS tests, the build with `tsc` against `@types/node` 24.19.2 | all pass | **pass**: 10, 3 and 66 tests; both apps build (06) |

Also run locally on `9d5180a`, not pictured: `just pre-push` (the workspace's 790 Rust tests, the JS suites, guards and the secret scan) and `just audit` (advisories, bans, licences, sources, the JS licence policy and `pnpm audit`). Both pass.

## Not validated, and why

- **"Split via Rust" itself, the one call from the screen into Rust.** It goes through Tauri's IPC, which exists only inside the Tauri window, not in a browser. Opening that window on the owner's desktop and recording the screen was not done.
  - The command's Rust side is untouched by this pull request.
  - Its JavaScript side, `invoke` from `@tauri-apps/api` 2.12.4, is only type-checked by the build here. No test exercises it yet: the terminal's tests deliberately don't render `App`, because it needs an IPC mock (`screens/Sale.test.tsx` says why).
  - That call path with 2.12.4 is unverified by this evidence.
- **The desktop bundles:** CI builds them for macOS and Windows on promotion only.
