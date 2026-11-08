# Validation: 1.8.3's third part, the finalize

**Commit validated:** `7d86d4f`, the pull request's head. It was checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **`sale_finalize`** (new, `commands::sale::finalize`) completes the sale the till is paying for, in one transaction:
  - the acting user read again;
  - the receipt number;
  - the completed sale, and its receipt rendered for the register's profile;
  - one commit, its manifest and delivery rows, before any guarded fact;
  - the sale's rows, the stock events, the original and its print job, the `sale.finalize` audit row, the journal's removal and the basket's consumption;
  - the completion, last.

  It refuses by name, before anything is written, what could never complete. The printer and the drawer run after the commit only.
- **`Register::receipt_device`** (new): every register renders for the 80 mm simulator profile with no device until 1.7.14, as the owner decided on 8 November 2026.
- **`StoreRepository::profile`** and **`TenderTypeRepository::kinds`** (new, `pos-db`).
- **The shell's audit helper** sends each row's defined payload in its envelope member instead of `{}`.

The runtime is shown through a harness linked against this head's `terminal`, `pos-db`, `pos-domain` and `pos-hardware`. No screen calls the finalize yet: its screen is 1.11.7's.
- **Each harness step is a new process.** It opens the register with `Register::open_with` on a file built by `pos_db::open`, as `run` opens it, and drives the shipped commands: sign-in, cart, tender, finalize.
- **The printer scenarios (03–05)** attach a receipt device to the register through `with_receipt_device`: 1.7.10's network driver on the 80 mm profile over TCP, printing to 1.7.12's emulated printer. A drawer is on the emulator's port, and both are started in the harness's process on `127.0.0.1`. The pages on boards 03 and 04 are the emulated printer's own renders of the bytes it received.

**How it ran:**
- **The suites (01):** run in the worktree:
  - the `Done when`, `cargo nextest run -p terminal commands::sale::tests::`;
  - the store and tender-kind suites;
  - the tender suite's audit-payload test and the PII trace;
  - every `terminal` test;
  - clippy over `terminal` and `pos-db` with `-D warnings`;
  - the frontier check.

  Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–06):**
  - Built `--offline` against the worktree, with a fresh synthetic register file per scenario.
  - Each step's exit status is recorded beside its output.
  - A step written `!finalize` must be refused.
  - Where a scenario changes the register's file between processes, it says so: the store's GST registration, its fiscal decision, its receipt language.
  - The board builder refuses to render an unexpected exit, a failed check, the fixture's PIN or a local path.
- **The planted defects (07):**
  - 29 of them, run at this head in the same worktree before the pull request opened.
  - Each was run against the suite that names its rule, then restored with git.
  - No test and no migration was mutated, and the worktree was clean afterwards.
- **Builds ran without debug information** (`CARGO_PROFILE_DEV_DEBUG=0`). That changes nothing a test observes.

**Data is synthetic throughout:**
- the reference chain's fixture merchant, "Registered Chain Fixture", with its store, exempt from e-invoicing with its evidence and registered for GST;
- register `REG01`, with its shift open on 2026-11-08;
- cashier C-100, "Synthetic Cashier", whose PIN is typed by the harness and printed nowhere;
- two `SYN-` products.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal commands::sale::tests::` | 11 pass: eight named, three of the finalize's own | **pass**: 11 of 11 (01) |
| 2 | The store and tender-kind suites; the audit-payload test and the PII trace | all pass | **pass**: 6 of 6; 2 of 2 (01) |
| 3 | Every `terminal` test, clippy, the frontier | all pass, clean, reconciled at 124 | **pass**: 102 of 102; clippy clean; the frontier reconciles (01) |
| 4 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `7d86d4f`: 977 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog reconciles. The recipe stopped after the gate, at its fetch: the network was down. The branch was pushed and the pull request opened by hand at the same head (01) |
| 5 | Oil and 101 g of cheese, paid 1.000 then 5.000, finalized with no printer attached | completed and numbered; stock moved; one ready commit naming every fact; journal and basket gone; receipt queued; drawer `no_device` | **pass**: `REG01-000001`, total charged 2.110, change 3.890; oil 10.000 to 9.000, cheese 0.500 to 0.399; one ready commit, 15 members, 15 delivery rows; no journal, no basket row; the original queued, 61,213 bytes for `simulator-80mm`; the audit row records `drawer: requested` and travels as its defined payload (02) |
| 6 | The same sale on a register printing to the emulated printer | printed after the commit, byte for byte the stored original; the drawer opens | **pass**: receipt `Printed`, job `printed`; the bytes received equal the stored original; a 576 × 850-dot page, cut; drawer open (03) |
| 7 | The same sale at a store whose receipts are set in English | the store and each tender named in English | **pass**: "Synthetic Store", `SALE`, `Cash`, `Change` (04) |
| 8 | The emulated printer out of paper | the sale complete; the receipt refused and queued for its retry; the drawer kicked once | **pass**: receipt `Refused`, job `failed` with one attempt; drawer open; one completed sale (05) |
| 9 | The emulated printer silent 64 bytes into the receipt | the sale complete; the drawer kicked once | **pass** for the sale and the drawer. **Finding:** the receipt is reported `Printed` while the emulator printed no page, explained below (05) |
| 10 | Finalize a sale still owed 0.250; then a store not registered for GST; then one with no evidenced fiscal decision | each refused by name, nothing written, the journal still `tendering`; with the store restored, the sale completes | **pass**: `sale_not_settled`, `store_not_gst_registered`, `fiscal_obligation_undecided`, with the ledger unchanged each time; then `REG01-000001` completes (06) |
| 11 | 29 planted defects | each turns the test that names its rule red | **pass**: 29 of 29 caught, each compiled (07) |

`protected-paths` is red by design. It names exactly one path: the one-line `PLANNED` retirement in the byte-pinned `scripts/check-test-catalog.py`, which needs the owner's reading. Every other required check except `validation` is green or still running at this head.

## Found, and not changed here

**A network printer that goes silent after accepting a receipt is reported as having printed it.** Board 05's second scenario shows it. 1.7.10's driver returns `Printed` once the document's bytes are written to the socket. A 61 KB receipt fits in the operating system's socket buffer, so a printer that stops reading after accepting it is never noticed. Its job closes `printed`, nobody is asked whether paper came out, and no duplicate is offered. 1.7.10's own test reaches `Unknown` only with a 32 MB document, larger than any buffer.

The finalize reports what its device answers: for a device that answers `Unknown`, `hardware_failure_after_commit_leaves_sale_complete` shows `Unconfirmed`. Asking the device's status after the document, and treating no answer as `Unknown`, is the driver's fix. It is raised to the owner for 1.7.14 or a 1.7.10 follow-up.

## Not validated here, and why

- **No screen calls the finalize.** Its tender screen is 1.11.7's. The harness calls the same plain function the Tauri command wraps.
- **No real printer.** None is bought before milestone 5.8. Every printed page here is the emulated printer's.
- **A restart mid-finalize** is 1.8.4's recovery half, which this pull request leaves for it. Here the replay is shown in-process: `finalize_is_atomic_under_injected_failure` fails the finalize at each of its 12 write points, then completes the sale on the next call.
