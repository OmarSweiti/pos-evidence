**Runtime change: yes.** `pos-hardware` gains a conformance test target and a cross-check, `infra/emulators/emupos/` gains a pinned image, and CI gains a job. Nothing a register ships changes, but the change touches `crates/`, so it is validated as runtime: the suite against the emulator for every profile, the pinned image built and judged, mutants planted, and the same job read back from GitHub's runner.

**Setup**
- **Code:** PR head `9be94fd` with a clean tree, in a scratch worktree. It holds 1.7.13's six commits, merged with `development` after #276.
- **The pinned emupos:** built from this commit's `infra/emulators/emupos/`. Its base is `python:3.13-slim` by digest, it installs 26 packages from 491 locked hashes, and it runs as uid 10001. emupos's own wheel is the `d41f15ca…` 1.7.12's validation verified. It was started on `127.0.0.1` only.
- **Mutants:** run in the same worktree, so the repository was never edited.
- **Screenshots:** headless Chrome with a throwaway profile, each rendered from the run's own files. Local paths are made relative to the repository.
- **Data:** synthetic only.

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p pos-hardware --features emulator --test conformance` | the suite passes for every receipt-printer profile | ✅ 3/3 in 19 s, for 8 profiles. The TM-T20III, the one fully sourced file, prints dot for dot, takes 4607 rows in strips of at most 2303, refuses 2304, and refuses paper-out, cover-open and offline before a byte. The other files print nothing and name only what they do not know. The two USB files are refused by the network driver before a connection (01) |
| 2 | The device target | refuses unless both `HW_DEVICE_ADDR` and `HW_PROFILE` name one device | ✅ seven refusals, including a hostname, `0.0.0.0` and port 0 (02) |
| 3 | The synthetic capture | replays byte for byte | ✅ 5 exchanges. Its own fields say it is synthetic (02) |
| 4 | The pinned emupos, from `infra/` | builds, runs as non-root, and reports healthy | ✅ emupos 0.4.0, 26 packages, uid 10001, `{"status":"ok"}` (02) |
| 5 | The cross-check | every stream dot-identical; status agrees | ✅ the receipt as `GS v 0`, 4607 rows in `GS v 0` strips, and the receipt as `GS ( L` strips, all dot-identical. `DLE EOT` 1–4 agree in ready, paper-out, cover-open and offline (02) |
| 6 | Five mutants of the driver and the emulator | each caught | ✅ four by the suite: paper-out let through at pre-flight, a raster one row over the limit, the drawer pulse rounded down, and a ready printer answering `16`. The capture also catches the last of these. The fifth, an emulator that prints against an unknown framing, **survives the suite**: no seeded file isolates that guard. It is caught by the isolating case this PR adds to 1.7.12's emulator test ("a page against an unknown framing") (03) |
| 7 | A shared misreading: the encoder inverts every raster byte, and our emulator inverts it back | our own suite is blind to it; the independent emulator is not | ✅ the suite **passes** 3/3. The cross-check **fails**: 650,880 dots differ, every dot of the 576 × 1130 receipt. Our encoder's own unit tests also catch this particular one (8 fail). The suite cannot, and that blindness is what §6b.8's second reader exists for (04) |
| 8 | `printer-conformance` on GitHub's runner, at this head | green | ✅ every profile run, the image built, emupos healthy, all three streams dot-identical, and status agreeing in all four states (05) |
| 9 | `just pre-push` on this head | green | ✅ exit 0 in 127 s from a clean tree: 469 Rust tests (4 skipped: the two ignored tests here and two existing), 79 web tests, lint, guards, build-web, and the full-history secrets scan |

**Seen during the run.** The evidence pass took 41 minutes rather than about 3, during an internet outage on this machine. Its results match an earlier pass at `34acc9f`, which differs from this head only by the isolating test, line for line. In the fifth mutant's run, nextest also reported one emulator test as *leaky*. That was under a mutant and an outage, and no clean run here showed one.

**Not validated here, and why**
- **The golden replay.** `every_golden_renders_dot_identical_through_the_independent_emulator` needs 1.7.5's goldens. 1.7.13 is partially delivered, as its `Full-step status:` marker says.
- **A physical device.** The device target's own test is ignored until the gate (5.8.3).
- **`Partial` against the emulator.** The emulator has no fault that closes mid-document, so 1.7.10's fake proves it.
- **Merge enforcement.** `printer-conformance` is not a required check yet, so a red one would not block the merge button.
