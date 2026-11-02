# Validation: 1.6.7, the IPC command registry and its exhaustiveness test

**Commit validated:** `faedc6e`, the pull request's head, checked out clean in a scratch worktree with its own build directory.

**What changes at runtime.** One line of the shell: `run` now installs `invoke_handler()`, which `commands![split_tender]` generates, in place of writing `tauri::generate_handler![split_tender]` inline. The rest of the change is the registry, a table and a check that nothing in the running app calls, plus tests and documents.

**How it ran:**
- **The `Done when` and the suites.** The `Done when` ran, then the `terminal` crate, clippy over it, and the whole workspace (01).
- **The harness.** A harness linked against this head's `terminal_lib` and `pos-domain`, built `--offline` from the worktree's `Cargo.lock` (02 to 04).
- **The expansion.** The compiler's macro expansion of the shell's crate at this head, `-Zunpretty=expanded` (02).
- **The planted defects.** 17 defects, each applied to the worktree, tested and restored (05).

Data is synthetic throughout: invented ids, an invented department and an invented policy. No database is opened: the registry reads none, and the threshold runs through the pure domain.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal --test ipc_contract` | 5 tests pass | **pass**: 5 of 5 (01) |
| 2 | The `terminal` crate, clippy `-D warnings`, the workspace | all pass | **pass**: 39 of 39; clippy clean; 795 passed, 4 skipped, the existing `#[ignore]` ones (01) |
| 3 | `REGISTERED` and `COMMANDS` at this head, then `check` | `["split_tender"]`; one row that proves no capability and gives its reason; `Ok(())` | **pass** (02) |
| 4 | The handler `run` installs, as the compiler expanded it | its one match arm is `"split_tender"`, every other name returns `false`, and it parses the row's two declared arguments | **pass**: arms `"split_tender"` and `_ => return false`; keys `totalMinor` and `parts`; `run` calls `.invoke_handler(invoke_handler())` (02) |
| 5 | Two privileged rows from the contract, as written | accepted | **pass** (03) |
| 6 | Each row altered in one place | refused by name, with a readable message | **pass**: 17 refusals, one per alteration (03) |
| 7 | The department threshold through `add_department_line`, policy 10.000 cap, 5.000 threshold | 5.000 passes alone and consumes no offered approval. 5.001 is refused alone, passes with a matching handle, and is refused with a handle for another amount or sale. Nothing passes the cap | **pass**, all ten cases (04) |
| 8 | 17 planted defects | each turns the test that names its rule red | **pass**: 17 of 17 caught, every mutant compiled, worktree restored with no changed file (05) |

Also run locally on the branch, not pictured:
- `just pre-push`, run by `just pr` at `a7111cb`: lint, the test catalog and frontier reconciliation, tests, the web build, guards and the secret scan. It passed.
- At `faedc6e`, which changes only the wording of one refusal: `cargo fmt`, clippy over the crate, and its 39 tests.

## Not validated, and why

- **The IPC round trip in the app window.** Tauri's IPC exists only inside the desktop window, and opening it on the owner's desktop was not done. In its place, the compiler's own expansion at this head (02) shows the handler `run` installs: it answers `split_tender`, with the arguments the registry declares, and nothing else.
- **A declaration against every handler's real signature.** No command but `split_tender` is registered, and its two arguments are checked by eye in the expansion (02). Doing this mechanically needs the generated TypeScript schema, which doesn't exist yet. The phase file leaves that to the step whose `Done when` reads it (1.4.11).
- **The database's refusal of an already-consumed approval.** That is 1.4.11's `Done when`, with the command that consumes it.
