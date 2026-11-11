# Validation: 1.7.14a, the printer profiles shipped with the app, and the binding's one door

**Commit validated:** `a984d6e`, the pull request's head. It was checked out clean in a scratch worktree with its own build directories.

**What changes at runtime:** in `pos-hardware`, three things a register will use when it binds its printer (1.7.14b).
- **`profile::data::SHIPPED_PRINTERS`:** the eight receipt-printer profile files, compiled in.
- **`profile::data::printer(profile_id, version)`:** answers the profile a binding names, or refuses it as `NotShipped` or `Unresolved`, naming every parameter the driver needs that it leaves unknown.
- **`PrinterProfile::emulator(paper)`:** the simulator's values reached over TCP 9100. The emulator suite now uses it.

Nothing in the register calls `printer` yet, which is 1.7.14b's, so there is no screen to drive. The runtime is shown through a harness linked against this head's `pos-hardware`, built `--offline` in its own build directory, with 1.7.12's emulator.

**How it ran:**
- **The suites (01):** run in the worktree: the profile and emulator suites, every `pos-hardware` test with every feature, and clippy with `-D warnings`. Then `just pr`'s gate at this same head is quoted from its log. Its test run passed 1037, with one existing `pos-db` auth test flagged leaky by nextest, unrelated to this change.
- **The harness (02–04):** `door` asks the door for every profile this build carries and three it does not. `shipped` compares each compiled-in copy with its file. `print` asks the door for `emulator-80mm`, starts the emulator with it, and prints the suite's synthetic receipt through the shipped network driver. The board builder refuses a local path, a UUID, a failed check, or an outcome other than the one written.
- **The planted defects (05):** 6 of them, run at this head before `just pr` against the profile and emulator suites, each restored with git. The worktree was clean afterwards.

**Data is synthetic throughout:** the profiles are the repository's own seed files. The receipt is the suite's synthetic one, with its buyer block and fiscal id left out so the image shows no buyer and no id.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | The profile and emulator suites | all pass, the two new tests among them | **pass**: 12 of 12 (01) |
| 2 | Every `pos-hardware` test; clippy | all pass; clean | **pass**: 115 of 115, 2 skipped (existing); clippy exit 0 (01) |
| 3 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `a984d6e`: 1037 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog and the frontier reconcile (01) |
| 4 | The door, for the simulator's and the emulator's profiles | answered, each knowing every parameter the driver needs; the emulator's over TCP 9100 | **pass**: four `BOUND` lines; `emulator-80mm v1: transport Tcp9100` (02) |
| 5 | The door, for each of the eight shipped files | each refused, naming exactly what it leaves unknown | **pass**: eight `REFUSED … unresolved` lines, every one naming its cut, its feed, its drawer pulse and its Arabic support among them (02) |
| 6 | The door, for an id or version the build does not carry | not shipped | **pass**: three `not shipped with this build` (02) |
| 7 | The compiled-in copies against the files | each equal, byte for byte; as many as the directory holds | **pass**: 8 of 8 equal; 8 files (03) |
| 8 | `emulator-80mm`, through the door, to the emulator | status read; the receipt printed, cut, dot for dot | **pass**: `Ready`; `Printed`; `cut: true`, `dot for dot what was sent: true`; the emulator's rendering shown beside (04) |
| 9 | 6 planted defects | each turns the test that names its rule red | **pass**: 6 of 6 caught, each compiled (05) |

## Not validated here, and why

- **No register binds yet.** The binding read at open, the alarm when it cannot be honoured, the worker and the drawer's answer are 1.7.14b–d, each with its own evidence. 1.7.14's `Done when` is theirs.
- **No real printer.** None is bindable before milestone 5.8, which is what step 5 shows.
