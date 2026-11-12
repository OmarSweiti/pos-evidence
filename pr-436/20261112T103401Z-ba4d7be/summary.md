# Validation: #436, 1.7.14d, the drawer's answer is `pulse_sent`, never `opened`

Validated at `ba4d7be`, the pull request's head, in a clean scratch worktree, against 1.7.12's emulated
printer (`pos_hardware::emulator`) under the `emulator-80mm` profile. The register file is the shell
suites' own synthetic shop, built fresh through `pos_db::open` for every run. There is no screen yet, so
the app is driven through its shipped commands by an evidence-only harness linked against the worktree's
own crates. The harness is not part of the repository.

## Commands

- `cargo nextest run -p terminal --test printer_binding`: once in full and five times more.
- `cargo nextest run -p terminal --lib commands::sale::tests::`
- `cargo nextest run --no-fail-fast -p terminal -p pos-db -p pos-hardware --all-features`
- `cargo clippy -p terminal --all-features --all-targets -- -D warnings`
- `ev-d-harness <opens|absent|offline>`: built twice from one source, against `development` at
  `aa9b14a` and against `ba4d7be`.
- `just pr`'s gate (`just pre-push`) at `ba4d7be`, quoted from its log.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | 1.7.14's `Done when`: `printer_binding` against the emulator | 6 pass, including the four named tests and this part's own | 6 of 6 passed, and again in each of 5 more runs (board 01) |
| 2 | This part's unit test and the finalize's own | All pass, including `the_drawers_answers_say_no_more_than_the_driver_knows` | 12 of 12 passed (board 01) |
| 3 | Every `terminal`, `pos-db` and `pos-hardware` test; clippy | All pass; clippy clean | 623 passed, 2 skipped; clippy exit 0 (board 01) |
| 4 | `just pr`'s gate at this head | Lint, test, build-web, guards and secrets pass | 1045 passed, 4 skipped; every gate line passed, and the frontier reconciles 124 (board 01) |
| 5 | A sale with a drawer that opens plugged into the port | Before: `opened`. After: `pulse_sent`. One pulse; the drawer opened | As expected (board 02) |
| 6 | A sale with nothing plugged into the drawer port | Before: `opened`, which is wrong. After: `pulse_sent`. One pulse; nothing opened | As expected (board 02) |
| 7 | A sale with the printer reporting itself offline over `DLE EOT` | Before: `opened`, which is wrong. After: `pulse_sent`. One pulse; nothing opened | As expected: the printer reported `Offline` (board 02) |
| 8 | Six planted defects in the drawer's answer, each against the suites | Each turns its named test red, then is restored | 6 of 6 caught, every mutant compiled, the worktree restored to 0 changed files (board 03) |

## Not validated, and why

- **A real cash drawer and its sensor.** No device is bought before milestone 5.8. The emulator models a
  drawer's pin and solenoid voltage, and reports whether a pulse would open it. Nothing in the product
  reads a drawer back, which is the point of this change.
- **A screen showing the answer.** None exists until 1.11.x. The answer is shown as `sale_finalize`'s wire
  JSON.
