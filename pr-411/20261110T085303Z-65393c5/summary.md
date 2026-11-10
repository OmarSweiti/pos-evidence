# Validation: 1.7.6, the printer read at Pay, before any money is taken

**Commit validated:** `65393c5`, the pull request's head. It was checked out clean in a scratch worktree with its own build directories.

**What changes at runtime:** `tender_begin` reads the register's printer before it takes the till's lock. `PaymentSnapshot.printer` carries what it read: `ready`, `paper_out`, `cover_open`, `offline`, `no_status` or `no_printer`.
- **The commands that take money do not read it again** (`null`).
- **The reading is told, never enforced:** a sale whose printer would not print still completes, and its receipt is kept in the queue.

The runtime is shown through a harness linked against this head's `terminal`, `pos-db`, `pos-domain` and `pos-hardware`.
- **Each scenario is a new process** on a fresh register file, the shell suites' own shop, opened with `Register::open_with` as `run` opens it.
- **It drives the shipped commands:** sign-in, the cart, `tender_begin`, cash and the finalize, through the plain functions the Tauri commands wrap.
- **The printer is the real driver against the emulator:** 1.7.10's `NetworkPrinter` on the 80 mm profile over TCP, talking to 1.7.12's emulated printer, started in the same process on 127.0.0.1. Each scenario's fault (paper out, cover open, offline) is scripted onto the emulator's next connections.
- **Under each run, what the emulated printer saw:** every connection, every `DLE EOT` status question and its reply, every page and every drawer pulse.

**How it ran:**
- **The suites (01):** run in the worktree:
  - 1.7.6's `Done when`: `cargo nextest run -p pos-hardware at_pay` and `cargo nextest run -p terminal --test tender`;
  - every `pos-hardware` and `terminal` test;
  - clippy over both crates with `-D warnings`.

  Then `just pr`'s gate at this same head is quoted from its log.
- **The harness (02–03):**
  - Built `--offline` against the worktree, with the emulator feature, in its own build directory.
  - The board builder refuses any of the following: a non-zero exit, an unexpected refusal, the synthetic PIN, a UUID, or a local path. It also requires every reading named below.
- **The planted defects (04):** 9 of them, run at this head before `just pr`.
  - Each was run against the suite its rule belongs to, then restored with git.
  - The worktree was clean afterwards.
- **Builds ran without debug information** (`CARGO_PROFILE_DEV_DEBUG=0`).

**Data is synthetic throughout:** the reference chain's fixture store, with register `REG01` and a shift open on it dated 2026-11-05, cashier C-100 with one synthetic PIN, and ten bottles of oil at 1.250.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | 1.7.6's `Done when` | the reading's test and the tender suite pass | **pass**: 1 of 1; 13 of 13, two new (01) |
| 2 | Every `pos-hardware` and `terminal` test; clippy | all pass; clean | **pass**: 253 of 253, with 2 skipped (the device conformance target and CI's emupos cross-check, each `#[ignore]` by design); clippy exit 0 (01) |
| 3 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass** at `65393c5`: 1022 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog and the frontier reconcile (01) |
| 4 | A ready printer | Pay asks it and reads it ready; the sale prints | **pass**: `DLE EOT` 1–4 answered; `"ready"`, no warning, nothing taken; the cash answer `null`; `"printed"`, one page (02) |
| 5 | A printer out of paper, still out when the receipt is sent | `paper_out` at Pay with nothing taken; not read again; the sale completes and the receipt is kept for a retry | **pass**: `"paper_out"`, warned, 0 tenders; the cash answer `null`; `"refused"`; 1 receipt kept, 1 job failed for a retry; no page (02) |
| 6 | Out of paper at Pay, refilled before any money is taken | warned, back to the basket, read again ready, and the sale prints | **pass**: `"paper_out"`, then `"ready"`; `"printed"` (02) |
| 7 | Cover open; offline | each told at Pay | **pass**: `"cover_open"`, warned; `"offline"`, warned (03) |
| 8 | A device that answers no real-time status | never asked; no warning; it prints | **pass**: no status connection; `"no_status"`, no warning; `"printed"` (03) |
| 9 | No printer, as every register ships today | `no_printer`, warned; the sale completes and the receipt waits | **pass**: `"no_printer"`, warned; `"queued"`; 1 receipt kept, 1 job queued (03) |
| 10 | 9 planted defects | each turns the test that names its rule red | **pass**: 9 of 9 caught, each compiled (04) |

## Not validated here, and why

- **What the screen says for each reading.** The sale screen is 1.11.5's, with its own `paper_out_warns_at_pay_not_after`.
- **The printer-unavailable alarm, and a queued receipt printing once a printer returns.** Both are 1.7.6b's, next.
- **A real printer.** No device is bought before milestone 5.8 (`ref/hardware-and-receipts.md` §6b), and no register is bound to one before 1.7.14. The emulator answers `DLE EOT` with Epson's status bits.
