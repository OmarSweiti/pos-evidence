**Validated at `568b9d1`.** A one-line lockfile bump: `ctutils` 0.4.2 → 0.4.4, for RUSTSEC-2026-0385.

**Setup**
- **Advisory check:** `cargo deny check advisories`, the check CI's `supply-chain` job runs, over the lockfile on `development` (`58dbc6e`) and over this head.
- **Smoke test:** this exact commit, taken with `git archive`, built with `cargo build --locked -p pos-server` inside a `rust:1.97.1-slim-bookworm` container. It ran beside the project's own pinned `postgres:18-alpine` image (the digest in `infra/docker-compose.yml`) on a private Docker network.
  - **Isolation:** no host port was published. The host's 5432 and 8080 belong to another project, and `pos-server` binds a fixed `127.0.0.1:8080`, so every request was made inside the server's own container.
  - **Credentials:** the database password was random, never printed, and replaced with `<password>` in the transcript.
  - **Clean-up:** the containers and the network were removed afterwards.

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo deny check advisories` on `development` | the advisory refuses `ctutils` 0.4.2 | ✅ `error[vulnerability]` RUSTSEC-2026-0385, `advisories FAILED`. Board 01 |
| 2 | The same check on this head | clear | ✅ `ctutils` 0.4.4, `advisories ok`. Board 01 |
| 3 | Build `pos-server` from this commit with `--locked` | builds against 0.4.4, through `digest` 0.11 → `hmac`/`sha2`/`md-5` → `sqlx-postgres` | ✅ Board 02 |
| 4 | PostgreSQL's password setup | SCRAM-SHA-256 for TCP connections | ✅ `password_encryption scram-sha-256`, `pg_hba` rule `host all all scram-sha-256`, verifier stored as `SCRAM-SHA-256…`. Board 02 |
| 5 | `GET /health` and `GET /health/db` with the right password | both 200; the database reachable | ✅ `{"db":"ok"}`, and PostgreSQL logged `connection authenticated: identity="pos" method=scram-sha-256`. Board 02 |
| 6 | `GET /health/db` with a wrong password | 503; the login refused | ✅ `{"db":"error"}`, and PostgreSQL logged `password authentication failed for user "pos"`, so the check in step 5 was real. Board 02 |

CI at this head: `rust`, `printer-conformance`, `guards`, `web`, **`supply-chain`**, `protected-paths`, `topology`, `label`, and CodeQL with its five analyses are green, with **0 open code-scanning alerts**.

**Not validated here, and why**
- **Constant-time behaviour itself.** That is the advisory's fix, verified upstream with BINSEC (per the advisory). This PR proves only that the fixed release builds and that the authentication path it sits under still works.
- **The register.** `pos-db` and the terminal don't depend on `ctutils`, so no register code path changes.
