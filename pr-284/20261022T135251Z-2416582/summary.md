**Validated at `2416582`.** Microstep 1.2.5, migration `0007`, Arabic search and scale commissioning, is complete with this PR.

**Setup**
- **Code under test:** a clean scratch worktree of this exact head.
- **Harness:** throwaway, outside the repository, built against that worktree's `pos-db` with its `fixtures` feature.
- **Database:** a fresh encrypted register database. Its catalogue was created at schema version 6, then `pos_db::open` migrated it to 7 the way a register upgrading in the field does, so the backfill is exercised, not only the triggers.
- **Synthetic data only:** eight invented products, one invented barcode, one invented scale with a made-up evidence reference.
- **Every search goes through this head's `pos_db::search`:** the typed text is put through `exact` and `fold`, then sent to FTS5 as `name_ar_exact : "…" OR name_ar_fold : "…"`, which is the query 1.2.3's `search()` will build. Results are read back from the database in rank order.
- **No app, scanner, scale or emulator:** this PR is the schema and the query-side fold. Nothing on screen calls search until 1.2.3 and the sale screen.
- **The migration file itself is never edited,** not even in a scratch worktree, because the repository's immutability guard covers committed migrations. Three of its triggers are instead proved load-bearing by dropping each from a scratch database of its own (steps 16–18).

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | A v6 catalogue of 8 products is migrated | schema version 7; all 8 indexed by the backfill | ✅ Board 03 |
| 2–3 | Type `قهوة` with or without tashkeel | both coffees, one of whose names is fully vocalised | ✅ Board 01 |
| 4–6 | Type a hamza, alef maqsura or tatweel variant | `ابريق` finds `إبريق شاي`; `مستشفي` finds `حقيبة مستشفى`; `شاي` finds `شـــاي أخضر` | ✅ Board 01 |
| 7–8 | Two labnehs differ only by taa marbuta and haa | whichever spelling is typed, both are found and that spelling ranks first | ✅ `لبنة` → `لبنة` then `لبنه بلدية`; `لبنه` → the reverse. Board 01 |
| 9 | Type two characters, `قه`, as a prefix | both coffees, served by `prefix='2 3'` | ✅ Board 01 |
| 10 | Type one letter, `ه`, that sits inside `قَهْوَة` | nothing, because the raw vocalised name is not indexed | ✅ Board 01. Board 02 shows the two projections that are indexed. I also checked the reason outside the register, on SQLite 3.53.4: a table that does index the raw name with this migration's tokenizer breaks `قَهْوَة عَرَبِيَّة` into single-letter tokens only (eight distinct letters), which `ه` matches and `قهوة` does not. |
| 11 | English name, SKU and barcode | each finds its product | ✅ Board 03 |
| 12 | A barcode moves to a new product; a product is retired | the index follows both | ✅ Board 03 |
| 13 | A scale-label rule switched on before, then after, the scale's evidence says verified | refused, then accepted | ✅ Board 04 |
| 14 | The scale goes for maintenance | the rule switches itself off and cannot be switched back on | ✅ Board 04 |
| 15 | Edit a verification event; skip an event number | both refused: append-only, contiguous | ✅ Board 04 |
| 16–18 | Drop `barcode_au`, `trade_scale_disable_rules_on_status_loss`, `embedded_rule_requires_verified_scale_insert` from scratch databases | each statement it refused now lands, so each trigger is the one guarding it | ✅ all three. Board 04 |
| 19 | `verify-schema` and `verify-pg-migrations` | `0001`–`0007` apply as the register does; the mirror conforms | ✅ The Postgres mirror was applied to a throwaway PostgreSQL container: `DATABASE_URL` was unset, and no other server was touched. Board 05 |
| 20 | The tests (`Done when`) | `0007`'s suite, all of `pos-db`, and the terminal | ✅ 15/15, 189/189, 18/18. Board 05 |
| 21 | Three mutants of `pos_db::search`, the Rust half of the fold | each caught | ✅ Forgetting taa marbuta, stripping feh instead of tatweel, and folding spelling in `exact` each fail `prop_sql_and_rust_folding_agree` and at least one named search test. The worktree was restored to 0 changed files. Board 06 |

CI at this head: `rust`, `printer-conformance`, `guards`, `web`, `supply-chain`, `topology`, `label`, and CodeQL with its five analyses are green, with **0 open code-scanning alerts**. **`protected-paths` is red by design.** It names one file, `scripts/check-test-catalog.py`, where eight planned test names were retired.

**Not validated here, and why**
- **`search()` itself** is 1.2.3. The harness builds the query the way that repository will.
- **The scale-label parser against these tables** is 1.2.4's database half, which waits on this migration.
- **`plu_code`, `tile_grid` and `tile`** are created and applied by `verify-schema`, but no step writes them: the code that uses them comes later in Phase 1.
- **A real trade scale or JSMO verification.** None exists for this register. The evidence reference is synthetic, and embedded pricing stays switched off until a real one is recorded.
- **The server's search and scale tables.** The mirror creates nothing and records why: FTS5 is SQLite's, and the reference and fact tables wait on the unsigned multi-tenant decision.
