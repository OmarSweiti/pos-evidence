# Validation: 1.7.14b, a register reads its printer binding when the till opens

**Commit validated:** `53c9ff4`, the pull request's head. It was checked out clean in a scratch worktree with its own build directories.

**What changes at runtime:**
- **Opening the till reads the register's `printer.binding` setting** (a profile id, a version, an IP address and a port), through 1.7.14a's door.
- **A binding it can honour becomes the register's printer.** Every original receipt is rendered for that profile and printed at the address.
- **A binding it cannot honour opens the till with no device, never a closed till.** The four reasons are `profile_not_shipped`, `profile_unresolved`, `not_a_network_printer` and `setting_unreadable`. The register renders for the simulator's profile and keeps and queues every receipt.
- **`health_status` carries the binding.** A refusal raises `printer_binding_refused`, the loudest alarm.

The runtime is shown through a harness linked against this head's `terminal`, `pos-db`, `pos-domain` and `pos-hardware`, built `--offline` in its own build directory.
- **Each scenario is a new process.** It builds the shell suites' own shop file through `pos_db::open` and writes the register's binding row, as a settings screen will once one exists.
- **It then drives the shipped code:** it opens the till with `Register::open_with` as `run` does, signs in, reads `health_status`, and sells one bottle of oil for cash through the shipped commands.
- **The device is 1.7.12's emulated printer,** started on the loopback under the emulator's 80 mm profile.

**How it ran:**
- **The suites (01):** run in the worktree: 1.7.14's `printer_binding` suite against the emulator, the setting reader's suite, the guard that nothing a register ships enables the emulator, every `terminal`, `pos-db` and `pos-hardware` test with every feature, and clippy with `-D warnings`. Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–04):** one process per binding. The board builder refuses any of the following: a non-zero exit, a failed check, the synthetic PIN, the word `argon2`, a UUID, or a local path. It also asserts the key lines the steps below name.
- **The planted defects (05):** 8 of them, run at this head before `just pr`, each restored with git. The worktree was clean afterwards.

**Data is synthetic throughout:** the reference chain's fixture store with register REG01 and a shift open, ten bottles of oil, and the synthetic cashier C-100. The address `192.0.2.10` is TEST-NET-1, which nothing reaches.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | 1.7.14's suite; the setting reader; the emulator guard | all pass | **pass**: 3 of 3, the step's two named tests among them; 5 of 5; the guard passes (01) |
| 2 | Every `terminal`, `pos-db` and `pos-hardware` test; clippy | all pass; clean | **pass**: 619 of 619, 2 skipped (existing); clippy exit 0 (01) |
| 3 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `53c9ff4`: 1041 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test (01) |
| 4 | Bound to the emulator's 80 mm profile | the binding honoured; health ready with no alarm; the receipt rendered for `emulator-80mm` and printed, cut, on the emulator | **pass**: `bound`; `ready`, `[]`; `printed`, rendered for `emulator-80mm`, 0 waiting; the emulator's job cut, its rendering shown (02) |
| 5 | Bound to the TM-T20III's shipped profile | refused as `profile_unresolved`, naming its four unknowns; the till open with no device; the binding's alarm first; the sale complete, its receipt queued | **pass**: as expected, `["printer_binding_refused","printer_unavailable"]`, `queued`, rendered for `simulator-80mm` (03) |
| 6 | No binding | unbound: only the missing printer raised | **pass**: `unbound`, `["printer_unavailable"]` (04) |
| 7 | A profile not shipped; the simulator's (not on the network); a host name for the address | each refused by name; each till open and selling | **pass**: `profile_not_shipped`, `not_a_network_printer`, `setting_unreadable`; each `queued` (04) |
| 8 | 8 planted defects | each turns the test that names its rule red | **pass**: 8 of 8 caught, each compiled (05) |

## Not validated here, and why

- **No screen writes a binding yet.** A settings or set-up screen will, audited. Here the row is written as the tests write it.
- **Printing still happens in the finalize command.** The print worker off the command path is 1.7.14c's, and it completes 1.7.14's `Done when`.
- **CI's `supply-chain` check is red on this pull request** for a reason outside it: a new RustSec advisory in `rustix` (RUSTSEC-2026-0486) fails it for every pull request. It is fixed in its own pull request, and the check is re-run here once that lands.
