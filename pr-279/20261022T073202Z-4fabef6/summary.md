**Runtime change: yes.** The lockfile moves `tokio` from the yanked 1.54.0 to 1.54.1. `tokio` runs in the server, so the server was built and run on it.

**Setup**
- **Code:** PR head `4fabef6` with a clean tree, in a worktree.
- **Containers:** throwaway ones on an isolated Docker network, with nothing published to the host. Port 8080, where the server listens, is in use on this machine by another project, and was left alone.
  - `rust:1.97.1-slim` is the pinned toolchain. It builds `pos-server`, runs its tests, and runs it.
  - `postgres:18-alpine` is pinned by the same digest CI uses.
  - `curlimages/curl` joins the server's own network namespace to call it.
- **Data:** none; the throwaway database is empty.

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo deny --locked check advisories` on `development`'s lock | the failure CI reports | ❌ as expected: `error[yanked]: detected yanked crate (try cargo update -p tokio)`, `advisories FAILED` (01) |
| 2 | `cargo deny --locked check advisories bans licenses sources` on this head | passes | ✅ `advisories ok, bans ok, licenses ok, sources ok` (01) |
| 3 | Build `pos-server` with the pinned toolchain | builds, linking tokio 1.54.1 | ✅ `rustc 1.97.1`; `cargo tree -i tokio` gives `tokio v1.54.1` (01) |
| 4 | `cargo test -p pos-server` | passes | ✅ 4 passed (01) |
| 5 | Run it without `DATABASE_URL`, then `GET /health` and `GET /health/db` | 200 `ok`, and 503 `unconfigured` | ✅ `{"status":"ok",…}` → 200; `{"db":"unconfigured"}` → 503 (01) |
| 6 | Run it against the throwaway Postgres, then the same two calls | 200 and 200 `ok`: a real round-trip through `sqlx` on the new runtime | ✅ `{"status":"ok",…}` → 200; `{"db":"ok"}` → 200 (01) |
| 7 | CI on this head | every required check green | ✅ `rust`, `guards`, `web`, `supply-chain`, `protected-paths` and `topology` all pass; `supply-chain` was red on #278 for this yank |

**Not validated here, and why**
- **Windows.** The resolver re-picked which `windows-sys` six Windows-only dependents use, among versions already locked. Nothing here builds them; CI's cross-platform matrix runs on promotion.
- **The register.** It does not link `tokio`, so there was nothing on it to run.
