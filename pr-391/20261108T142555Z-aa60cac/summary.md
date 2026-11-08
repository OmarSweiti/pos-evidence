# Validation: 1.9.3, receipt numbering

**Commit validated:** `aa60cac`, the pull request's head. It was checked out clean in a scratch worktree with its own build directory.

**What changes at runtime:**
- **`commands::sale::next_receipt_number(tx, register, code)`** (new) takes a register's next receipt number inside the caller's transaction and prints it as `REG01-000123`: the register's code, a hyphen, and its own counter padded to six digits.
- **`Register::code`** (new) is the register's code, read with its row when the till opens.
- **`SequenceRepository::next_prefixed`** (new, `pos-db`) stores the prefix with a counter's first number and returns the stored one ever after.
- **Two new refusals:** `SequencePrefixInvalid` and `SequencePrefixMissing`.

No command calls the numbering yet; the finalize, 1.8.3's third part, will. So the runtime is shown through a harness linked against this head's `terminal`, `pos-db` and `pos-domain`:
- each step is a new process;
- each opens the register with `Register::open_with` on a file built by `pos_db::open`, as `run` opens it;
- each takes numbers through the shipped `next_receipt_number`.

**How it ran:**
- **The suites (01):** run in the worktree:
  - the `Done when`, `cargo nextest run -p terminal --test receipt_number`;
  - the counter suite, `cargo nextest run -p pos-db --test sequence`;
  - every `terminal` test;
  - the frontier check.

  Then `just pr`'s gate at this same head is quoted from its log. It covers clippy over the workspace, the test catalog, every test in the workspace, the web tests and build, every guard self-test and the history secret scan.
- **The harness (02–04):**
  - Built `--offline` against the worktree, with a fresh synthetic register file per scenario.
  - Each step's exit status is recorded beside its output.
  - A line written `!number` is a refusal the scenario expects.
  - The board builder refuses to render an unexpected exit, a failed check, a gap report that does not name the one missing number, or a local path.
  - Where a step changes the register file by hand, it says so: the register's code changed in its row, and parked sales carrying chosen numbers for the gap report.
- **The planted defects (05):**
  - 10 of them, each applied to the worktree.
  - Each was run against the suite that names its rule, then restored with git.
  - No test and no migration was mutated, and the worktree was clean afterwards.
- **Builds ran without debug information** (`CARGO_PROFILE_DEV_DEBUG=0`). That changes nothing a test observes.

**Data is synthetic throughout:**
- the reference chain's synthetic store;
- registers `REG01`, `REG02`, `REG1`, `REG11`, `REG 05` and `REG06`;
- the display name "Front till";
- device id `device-REG01`.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal --test receipt_number` | 2 pass | **pass**: 2 of 2 (01) |
| 2 | `cargo nextest run -p pos-db --test sequence` | 13 pass, three of them new | **pass**: 13 of 13 (01) |
| 3 | Every `terminal` test, and the frontier | all pass; the frontier reconciles | **pass**: 91 of 91; the frontier at 77 of 123 (01). nextest flagged one existing test, `print_queue::tests::a_retry_delay_of_zero_is_refused`, as leaky in that run. That test is pure, calling only `Timing::new` with no I/O, and this PR does not touch it. Three reruns of `print_queue::` reported no leak |
| 4 | `just pr`'s gate | lint, test, build-web, guards and secrets pass | **pass**: 962 passed, with 4 skipped (the existing `#[ignore]` ones); 79 web tests; every guard self-test; the test catalog reconciles; no not-ok, FAIL or error line (01) |
| 5 | A register coded `REG01`, named "Front till", opened as `run` opens it, takes two numbers | `REG01-000001`, `REG01-000002`; the prefix is the code, not the name | **pass**: both, and the counter row holds `next_value 3 · prefix "REG01-"` (02) |
| 6 | A number taken in a transaction that rolls back, then one that commits | the rolled-back number is not spent | **pass**: `REG01-000003` taken and rolled back with the counter unmoved, then `REG01-000003` committed (02) |
| 7 | The register's code is changed to `REG07` in its row, then it is opened and takes a number | the next number keeps the prefix the first number fixed | **pass**: opened with code `REG07`, issued `REG01-000004` (02) |
| 8 | Sales carrying numbers 1, 2 and 4; the gap report | the report strips the stored prefix and names 3 | **pass**: `GAPS [3]` (02) |
| 9 | Four registers of one store take numbers in turn, `REG1` a million receipts in | twelve distinct numbers; `REG1-1000001` beside `REG11-000001` | **pass**: 12 issued, 12 distinct; both present and different (03) |
| 10 | A register whose code holds a space; a receipt counter that already numbered without a prefix | each refused by name, no number spent | **pass**: `SequencePrefixInvalid` with no counter started; `SequencePrefixMissing` with the counter unchanged after the rollback (04) |
| 11 | 10 planted defects | each turns the test that names its rule red | **pass**: 10 of 10 caught, each compiled (05) |

Every required check except `validation` is green at this head.

## Not validated here, and why

- **The register app was not launched.** No command or screen numbers a receipt until the finalize, 1.8.3's third part. The harness calls the same public function the finalize will.
- **No printer emulator was used**, and no database server: nothing here prints, and `doc_sequence` is register-local.
