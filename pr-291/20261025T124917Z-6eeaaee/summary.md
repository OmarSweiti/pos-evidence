# Validation — microstep 1.2.8, the scan parser's fuzz target

**Commit validated:** `6eeaaee`, the pull request's head, checked out clean in a scratch worktree. No database, emulator or app is involved: the change is a fuzz target, its corpus, a test and a recipe. Inputs are synthetic codes only.

**How it ran:**
- `just fuzz parse_scan` and `just fuzz parse_scan 10000000`: cargo-fuzz 0.13.2 on the fuzz crate's pinned `nightly-2026-10-15`, on an Apple M3 laptop.
- `cargo nextest run` on the repository's stable 1.97.1.
- An evidence-only test, copied into the worktree for one run and then deleted, printed what each seed comes to.
- Each mutant edited one line of `crates/pos-domain/src/catalog/scan.rs` and was then restored. The run ends with "restored: 0 changed files".

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `just fuzz parse_scan` (Done when, first half) | exits 0 after 100 000 inputs, committed corpus unchanged | **pass**: `Done 100000 runs`, exit 0; 30 files and 0 changed files afterwards (01) |
| 2 | `cargo nextest run -p pos-domain --test parse_scan_fuzz` (Done when, second half) | 1/1 | **pass**: 1/1 (01) |
| 3 | `cargo nextest run -p pos-domain`, then `--workspace` | no failure | **pass**: 205/205; 574/574 with the 4 existing `#[ignore]` tests skipped (01) |
| 4 | Every seed, through the target's own `check` | each outcome matches its name; every refusal `parse_scan` can make is reached | **pass**: 30/30 match. Five plain, four price and two weight labels, and all seven refusals (02) |
| 5 | A longer search: 10 000 000 inputs | no finding, corpus unchanged | **pass**: exit 0 after 42 s; 236 edges, of which the seeds alone reach 234 (03) |
| 6 | Five one-line defects | the two layers together catch all five, and each catches one the other misses | **pass** (04): see below |
| 7 | Toolchain and licence facts | as stated in the PR | **as stated** (05): the stable pin refuses `cargo fuzz` (`-Z` is nightly only), and the nightly pin applies only inside the fuzz crate. The workspace's `cargo deny` passes. The fuzz workspace's, run by hand, fails licences on `libfuzzer-sys` (NCSA) |

### The five defects (step 6)

| Defect | Corpus test | Fuzzer |
|---|---|---|
| Span end added without an overflow check | caught | crash |
| Check-digit condition inverted | caught | crash |
| A label needs any digit, not all digits | caught | crash |
| A Tab trim that forgets to assign | **passes**: no seed ends in a Tab | timeout |
| An empty prefix claims every code | caught, by a seed's name | **nothing found**: the check reads the same layouts |

The third defect's finding (`2634\r\x007012504`) was then copied into the corpus. With the defect in place, the corpus test fails on that file; with the defect removed, it passes.

## Not validated, and why

- **The weekly fuzz job.** It doesn't exist yet; it belongs to 1.7.8b.
- **Any fuzz run on Windows or Linux.** Only macOS (aarch64) ran here.
- **The workspace lockfile.** It is unchanged; the fuzz crate's lockfile is separate.
- **CI's `cargo deny` over the fuzz lockfile.** CI doesn't run it. The by-hand result is shown in 05.
- **`protected-paths`.** It is red by design (the `justfile` is byte-pinned), as stated in the PR.
