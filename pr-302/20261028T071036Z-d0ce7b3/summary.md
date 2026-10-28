# Validation: microstep 1.3.7, the pinned tax pack and its import path

**Commit validated:** `d0ce7b3`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- `import_tax_pack`, `content_hash` and `TaxPackRepository::components_for` are database functions. No device, emulator or screen is involved, and no command calls them yet (`tax_rate_upsert` is Phase 3).
- A throwaway harness, not in the repository, called the pull request's own functions. Each run started from an empty register database opened through `pos_db::open`, with the same SQLCipher build the register ships.
- **Every pack is synthetic:** `SYNTH_*` band codes, a source that says it is not an official ISTD catalogue, dates in 2001, and rates that are arithmetic vectors. No real pack is seeded, because the merchant's bands are `ref/schema.md` §0003's open question.
- Each planted defect changed one statement of `crates/pos-db/src/tax_pack.rs` in the worktree and was restored. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-db --test tax_pack` (the `Done when`) | 11/11: the 5 the plan names and 6 more | **pass** (01) |
| 2 | The database crate, then the workspace | no failure | **pass**: 235/235; 646/646 with the 4 existing `#[ignore]` tests skipped (01) |
| 3 | Pin a synthetic standard pack; hash it again listed in reverse | the same pin either way | **pass** (02) |
| 4 | Import it with an approval dated 2001-01-15; read every rule back joined to its pack | each rule carries its real dates and the pack's version, source, pin, approval date and recording user; one pack behind every band, the standard band included | **pass** (02) |
| 5 | The same pack with one rate changed after pinning (16% → 17%) | refused as not the pack that was approved, nothing written | **pass** (02) |
| 6 | A standard store on the pack, by business date | 16% standard; reduced 4% to 30 June and 5% from 1 July; nothing before a band's first day; zero-rated and exempt at 0% | **pass** (03) |
| 7 | A store on a later version that drops the reduced band | the reduced band refused by name, never sold at another rate; standard still 16% | **pass** (03) |
| 8 | ASEZ: a zone pack with no standard pack, one missing a band, an ASEZ store on the standard pack, then a complete zone pack | the first two refused, the store refused by `0003`, the complete pack imported and its store charging the zone's 7%, not 16% | **pass** (03) |
| 9 | Thirteen malformed packs, each re-pinned after its change, and three that disagree with the register | each refused by name, with row counts inside the import's transaction unchanged | **pass**: 16 of 16, nothing written (04) |
| 10 | The `Done when`: the fixture against its enabled bands, then without each band, then without its approval | covers every band; not covered without any one; refused without the approval | **pass** (05) |
| 11 | Twenty planted defects | each caught | **pass** (06): all twenty caught, worktree restored |

## Found and fixed while preparing this validation

- **The refusal tests couldn't tell when a refusal was found.** The module says every refusal is found before anything is written. A refusal found after a write also leaves nothing behind, because the caller's transaction rolls back, so the first tests could not tell the two apart. They now count rows inside the import's own transaction, before the rollback. The planted defect "a rate for an unlisted band is found only after the writes" is caught that way (06). Fixed in `9874e6e`.
- **The read's approval check had no test.** No state the schema allows reaches it, because `0003` refuses to retire a pack a store uses. `a_store_on_a_retired_pack_is_refused_even_without_its_trigger` drops that trigger in its own scratch database and retires the pack, and the read still refuses. Added in `b79d0a4`.
- **An empty period's message said it ended "before it begins"** when it ends the day it begins. Corrected in `d0ce7b3`, the validated head.

## Notes

- In two of the twenty planted-defect runs, nextest reports one passing test as "leaky": a handle still open just after the test ended. It did not change any outcome. The runs at the head without a defect report none.
- The labeler gives a money-path risk label only to `pos-domain` tax files, so this pull request's `risk: money path` and `risk: compliance` labels were added by hand, matching its issue #301.

## Not validated, and why

- **The merchant's real pack**: the pinned official ISTD catalogue and the accountant's classification, which is `ref/schema.md` §0003's `⚠️ OPEN` item. 1.3.7 stays partially delivered until it arrives.
- **The return-box mapping and filing calendar** a pack must eventually carry (4.7.2), **Special Sales Tax**, **taxing a sale from the database** (1.8.3), and **the command that will import a pack** (Phase 3).
