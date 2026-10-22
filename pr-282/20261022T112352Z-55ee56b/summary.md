**Validated at `55ee56b`.** Microstep 1.2.4 is **partially delivered**: this PR is its pure half, the scan parser, `PriceSource`, `ScanLookup`, `resolve_scan` and the trade-scale gate, all in `pos-domain`.

**Setup**
- **Code under test:** a clean scratch worktree of this exact head.
- **Harness:** throwaway, outside the repository, built against that worktree's `pos-domain`. It plays one store's scans. The store has two scale layouts: price labels start `2`, weight labels `21`.
- **The shell's lookup** is played by a table of three products: a cola with a single and a six-pack barcode, and cheese and olives by PLU.
- **No database, emulator or app is involved.** This half is pure code: nothing here reads a disk, a clock or a device, and the shell that will is the deferred half.
- **Synthetic data only.** Every code, price and product is invented.

| # | Step | Expected | Observed |
|---|---|---|---|
| 1–2 | An ordinary barcode, then the same cola's six-pack outer code | the product at one unit, then at six: the code's own pack quantity (E.78) | ✅ Board 01 |
| 3 | A weight label from the deli scale | olives, 0.750 kg, at the catalogue price per kg | ✅ Board 01 |
| 4–5 | A price label printed at 09:00 (cheese 8.500/kg), scanned again at 21:00 after cheese rose to 9.900/kg | charged 0.416, the sticker, both times; only the advisory weight moves, from 0.049 to 0.042 kg (E.80) | ✅ Board 01 |
| 6 | The same label with one digit misread, so 0.416 reads as 0.916 | refused by the check digit, never charged as 0.916 (E.40) | ✅ `ChecksumFailed`. Board 01 |
| 7–8 | A label read one digit short; a label for an item nobody stocks | refused for its length; `Unknown`, so the queue offers quick-add (E.39) | ✅ Board 01 |
| 9 | A shell bug hands the cheese label a cola found by another code | refused: a label's price lands only on the product it names | ✅ `CandidateForAnotherCode`. Board 01 |
| 10 | A scanner on the wrong keyboard layout sends non-text bytes | a named refusal | ✅ `NotUtf8`. Board 01 |
| 11–15 | The scale behind the label: verified; awaiting maintenance; the day its evidence lapses; an untrusted clock; activating a rule with no evidence | allowed; then four refusals, each by name | ✅ Board 02 |
| 16 | What code outside the crate cannot do | it cannot build a price-embedded lookup, call the label constructor, deserialize a price or a lookup, or forge a parse | ✅ two compile-fail fixtures, each failing with exactly its golden errors. Board 03 |
| 17 | The tests: scan and scale, the compile-fail lane, the whole crate | all green | ✅ 19/19 (the three properties at 4,096 cases each), 1/1, 200/200. Board 04 |
| 18 | Sixteen mutants, each breaking one rule and then restored | every mutant fails at least one test | ✅ 16 of 16 caught, with 28 failing tests in all. The two that touch a derive or `#[non_exhaustive]` are caught by the compile-fail lane. The worktree was restored to 0 changed files. Boards 05–06 |

CI at this head: `rust`, `printer-conformance`, `guards`, `web`, `supply-chain`, `topology`, `label`, and CodeQL with its five analyses are green, with **0 open code-scanning alerts**. **`protected-paths` is red by design.** It names one file, `scripts/check-test-catalog.py`, where two planned test names were retired.

**Not validated here, and why**
- **The database half and the shell half.** They're deferred, as the PR body and 1.2.4's `Full-step status:` marker say. They wait on 1.2.5's tables and on 1.2.3's repository.
- **A scale label through the register.** No register path reads a scan yet; that path is the shell half.
- **One property of the mutant runs, recorded rather than hidden.** When a mutant breaks a property, proptest saves the failing input under `proptest-regressions/`, which is the crate's policy. Those inputs fail only against the mutant, so the script removes them after each one; the last line proves nothing else was left behind.
