# Validation: postgres:18-alpine at 10a5c69, with all three pins moved

**Commit validated:** `0e345fe`, the pull request's head, checked out clean. That's Dependabot's commit, `b5efbf3`, plus one commit on top of it.

**Runtime change: none.** The register and the server don't change. The pull request moves the pinned PostgreSQL image used by local development (`infra/docker-compose.yml`), by CI's Postgres service, and by the migration verifier. The evidence publisher classes workflow and compose files as non-runtime too.

**Why there's a second commit.** Dependabot moved only `infra/docker-compose.yml`. As `scripts/verify-pg-migrations.py` notes, that's all it ever does: the CI service image in `.github/workflows/ci.yml` and `PG_IMAGE` are moved by hand in the same change. `audit_image_pin` refused the half-applied bump, so `rust` was red and the engine pass was skipped. The second commit moves those two pins to the same digest.

## What was checked

| Check | Result |
|---|---|
| The new digest against Docker Hub | **pass**: it's the current multi-platform digest of `postgres:18-alpine`, pushed 26 October, past the 7-day cooldown (01) |
| What changed inside the image, both pulled by digest and run | the same PostgreSQL 18.6; the Alpine base moves from 3.24.2 to 3.24.3 (01) |
| The three pins | **pass**: all name `10a5c69…` (01) |
| `verify-pg-migrations.py --verbose`, with `DATABASE_URL` unset so it uses a throwaway container of the new image | **pass**: all 8 Postgres migrations applied to real PostgreSQL, and the container was removed (01) |
| `verify-pg-migrations.py --self-test` | **pass**: 70 cases (01) |
| The structural branch-workflow policy; CI-gate parity | **pass** (01) |
| `just pre-push`: lint, the workspace's 790 Rust tests, the JS suites, guards, the secret scan | **pass** (not pictured) |

## Red by design

`protected-paths`: `.github/workflows/ci.yml` and `scripts/verify-pg-migrations.py` are both byte-pinned policy files. Moving the pin through them needs the owner's review, which is why the bump lands by hand. This was reproduced locally against the pull request's base: "candidate changed trusted policy file .github/workflows/ci.yml; policy changes require an explicit red/manual security review".
