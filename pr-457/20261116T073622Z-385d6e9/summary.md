# 1.8.5c, part two: the till sets its store up (and 1.3.4's terminal half) — validation at 385d6e9

**Validated commit:** `385d6e9`, this pull request's head, from a clean scratch worktree.
**Runtime:** the till's two new commands, `store_setup_check` and `store_setup`, driven through the functions their handlers call (`store_setup::check`, `store_setup::set_up`) by an evidence-only harness linked against this head's `terminal` and `pos-db`. Every till is opened with `Till::open_in`, as `run` opens one; each harness verb is one process, so every start is a real one. There is no screen yet (1.11), so each answer is printed as the screen would receive it.
**Data:** synthetic only: `pos_db::fixtures::store_setup`'s file (`SYNTH_` bands, 2001 dates, a source that says it is not an official catalogue). The OS credential store is a stand-in file beside each scratch machine; this Mac's Keychain is never touched. Recovery codes go to paper files and are never printed. No PIN, PIN hash, key, code, id or local path appears in any image; the file's fingerprint is masked.
**Emulators:** none. The preview sale's receipt is not shown or inspected; receipts are 1.8.3's and 1.7's, and this change touches neither.

## Commands

- `cargo nextest run -p terminal --lib store_setup_` (1.8.5c's `Done when`), `--lib unapproved_tax_computation_policy_` (1.3.4's), and `cargo nextest run -p pos-db --lib store_setup::`, each once in full and three times more.
- The harness: `new`, `custody`, `file`, `check`, `apply`, `start <date>`, `unapproved`, `stopped`.
- `verify-audit --database <copy of register.db>` (debug build; `POS_DB_KEY` from the stand-in store).
- `just pr`'s gate (`just pre-push`) at this head, quoted from its log.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | Tests and gate (board 1) | 8 till tests (the four named among them), 1.3.4's test, 16 part-one tests pass every time; the gate green | All pass four times over; gate: 1134 workspace tests, lint, web builds, guards, secrets all green. CI's seven other required checks succeeded at this head. |
| 2 | Before custody (board 2) | A new machine refuses both commands as `store_setup_custody_incomplete`, nothing read or written | As expected; no file beside the (absent) database. |
| 3 | The check (board 2) | After custody, `ready` with the codes, 3 products, and each person with their role and 6 digits | As expected. |
| 4 | Refusals (board 3) | Unapproved policy, unregistered store, price above ceiling refused as data; a relative path refused; a short manager PIN refused as data naming staff number 2001 and 6 digits, not the PIN; a file changed after its check refused; still `store_not_set_up`, tables empty | As expected. |
| 5 | Set up (board 4) | `set_up`; device id file is the register row's; `store_set_up`; a new recovery code refused | As expected. |
| 6 | Started again (board 4) | Opens as the register, signed out; owner signs in, opens no sale; cashier types the business date a first start asks for and sells SYN-001 for 350 fils | Receipt REG01-000001, 350 fils. |
| 7 | Audit chain (board 4) | `verify-audit` verifies the copy | VERIFIED: 6 rows, intact; no anchor supplied, as the tool says. |
| 8 | 1.3.4 (board 5) | A second paid sale is refused at the finalize once its store's policy is not approved | `pricing_unavailable`, "the store's tax computation policy is not approved"; only the first sale completed. The policy was made unapproved on that scratch machine by dropping `0003`'s trigger, which no command can do. |
| 9 | A stopped setup (board 5) | Part one's write alone, no device id; the next start names the machine and opens | Device id written at start; `signed_out`. |
| 10 | Planted defects (board 6) | Each turns a test red | 21 of 21 caught at this head. |

## Not validated here, and why

- **The screen** that picks the file and asks for the PINs is 1.11's; the commands' answers are shown as data instead.
- **A real tax pack and approved policy** do not exist yet (`ref/schema.md` §0003's two `⚠️ OPEN` items); everything here is the synthetic fixture.
