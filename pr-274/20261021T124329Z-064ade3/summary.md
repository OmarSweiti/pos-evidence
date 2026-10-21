**Runtime change: yes.** `pos-hardware` gains its first driver that talks to a device: `NetworkPrinter` sends a document over TCP 9100, reads `DLE EOT` status back, kicks a drawer, and turns each socket event into one of 1.7.8's four outcomes. This validation drives it at the PR head over real TCP: to a third-party printer emulator, to a device that never answers, and to an address where nothing listens.

**Setup**
- **Code:** PR head `064ade3` with a clean tree. It is the merge of `development` (#272, #273) into the branch, whose 1.7.10 commit is `2950bbf`. No database is involved, because 1.7.10 touches none.
- **Harness:** a throwaway program outside the repository, linked against this head's `pos-hardware` and `pos-domain` through a scratch worktree of the head. It loads the committed profile files, renders the synthetic 60-line Arabic basket (576 × 5450 dots) and the short sample receipt, and calls only `NetworkPrinter`'s public API: `tcp`, `status`, `print` and `open_drawer`.
- **Emulator:** **emupos 0.4.0** as an **Epson TM-T20III** (`epson-tm-t20iii`, TCP 9100), wheel sha256 `d41f15caa2b286e7303b12c4bdcb02dc395c75553479d02b20566e2bacc2bb9a`. It ran in a throwaway `python:3.13-slim` container. Its printer port and control API were published on `127.0.0.1` only, and it started with no receipts. Faults were set and cleared through its control API. Its own event log was recorded inside the container from its event socket.
- **Mutants:** run in the same scratch worktree, so the repository was never edited.
- **Screenshots:** headless Chrome with a throwaway profile. 01 is emupos's own control page, open for the whole run. 02–05 are rendered from the run's own files: the harness's results, emupos's event log, the comparison, and the test logs.
- **Data:** synthetic only.

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | status() on emupos as a TM-T20III | Ready | ✅ Ready; emupos state {"drawer":"closed","faults":[]} (02) |
| 2 | print() the 60-line basket | Printed, and emupos completes one 576-dot receipt ending in a cut | ✅ Printed; emupos completed receipt "front-20261021T124144820Z-94f49b49" of 576 x 5450 dots, boundary="cut" (02, 04) |
| 3 | control: the short receipt sent raw (not through the driver) during paper-out | held while paper-out, then printed when it clears: the method below can see a leaked byte | ✅ receipts 1 → 1 while paper-out → 2 after clearing (02, 03) |
| 4 | paper-out set on emupos: status(), then print() the short receipt, then clear it | PaperOut; Failed(PaperOut); and nothing prints after clearing, so no document byte was sent | ✅ PaperOut; Failed(PaperOut); receipts 2 → 2 after clearing (02, 03) |
| 5 | cover-open set on emupos: status(), then print() the short receipt, then clear it | CoverOpen; Failed(CoverOpen); and nothing prints after clearing, so no document byte was sent | ✅ CoverOpen; Failed(CoverOpen); receipts 2 → 2 after clearing (02, 03) |
| 6 | offline set on emupos: status(), then print() the short receipt, then clear it | Offline; Failed(Offline); and nothing prints after clearing, so no document byte was sent | ✅ Offline; Failed(Offline); receipts 2 → 2 after clearing (02, 03) |
| 7 | open_drawer() with the seeded TM-T20III file, whose drawer_pulse is unknown | refused, naming drawer_pulse; emupos's drawer stays closed | ✅ Err("profile epson-tm-t20iii-ethernet-80mm does not know its drawer_pulse, which a register cannot be bound without"); emupos drawer "closed" → "closed" (02, 03) |
| 8 | open_drawer() with a harness pulse: Pin2, 100 ms on, 200 ms off | Ok; emupos opens its drawer, and its drawer.opened event reports the pin and 100 ms on (ESC p counts 2 ms units, rounded up) | ✅ Ok(()); emupos drawer "open" (pin and on-time: the event log, below) (01, 02, 03) |
| 9 | open_drawer() with a harness pulse: Pin5, 75 ms on, 150 ms off | Ok; emupos opens its drawer, and its drawer.opened event reports the pin and 76 ms on (ESC p counts 2 ms units, rounded up) | ✅ Ok(()); emupos drawer "open" (pin and on-time: the event log, below) (01, 02, 03) |
| 10 | NetworkPrinter::tcp() with the seeded TM-T88VII file (transport usb) | refused before any connection: wrong transport | ✅ Err("profile epson-tm-t88vii-80mm is reached by Usb, not by this driver") (02, 03) |
| 11 | status() as a TM-T20III against a listener that reads and never answers | Unsupported (neither ready nor offline), after asking DLE EOT 1 once | ✅ Unsupported after 501 ms; the listener received 10 04 01 (02) |
| 12 | print() the short receipt to the same silent listener | Printed: a device without status is not refused; it gets one DLE EOT 1, then the whole document | ✅ Printed; the listener received 89152 bytes: 10 04 01 1b 40 1d … (89152 bytes) (02) |
| 13 | the seeded Xprinter file (status_protocol none): status(), then print() the short receipt | Unsupported without a connection; the document arrives with no DLE EOT in front of it | ✅ Unsupported with 0 connection(s); Printed, the listener received 89149 bytes: 1b 40 1d 76 30 00 … (89149 bytes) (02) |
| 14 | print() and status() to 127.0.0.1:53612, where nothing listens | Failed(Unreachable); Offline | ✅ Failed(Unreachable); Offline (02) |
| 15 | Compare emupos's image of the basket with our bitmap | identical | ✅ 0 of 3,139,200 dots differ; 3 strips of 2303 + 2303 + 844 rows, 392,429 bytes (04) |
| 16 | `Done when`: `cargo nextest run -p pos-hardware net::` | exit 0 | ✅ 11 tests run: 11 passed, 86 skipped (05) |
| 17 | Four mutants of the driver | each fails its rule's test | ✅ the document in one write, not 16 KB pieces → `a_device_that_closes_mid_document_is_partial`; paper-out at pre-flight let through → `a_paper_out_pre_flight_fails_the_document_before_a_byte`; a device that never answers called offline → `a_device_that_never_answers_dle_eot_reports_status_unsupported`; the drawer pulse rounded down → `the_drawer_pulse_is_the_profiles_pin_and_timings`; the worktree restored with no change (05) |

14 of 14 driver calls behaved as specified. emupos's log also shows the drawer pulses arriving as `pin=2 on_time_ms=100` and `pin=5 on_time_ms=76`. The 75 ms request became 76 ms, because `ESC p` counts 2 ms units and the driver rounds up.

**Not validated here, and why**
- **Any physical device.** None is bought before milestone 5.8. This `Done when`'s device half (a golden printed, paper-out and cover-open read back, and the drawer kicked once on a TM-T20III and an SRP-332III) is 5.8.3's.
- **`Partial` and `Unknown` against emupos.** emupos cannot drop a connection partway through a document or stall a write. The in-process TCP fake proves both in the `Done when` (`a_device_that_closes_mid_document_is_partial` and `a_device_that_falls_silent_is_unknown`), and the one-write mutant shows the first test depends on the 16 KB pieces it claims to need.
- **A real drawer pulse.** No seeded profile knows one yet, and step 7 shows the refusal. Steps 8–9 use values the harness supplies; they are not sourced facts.
- **A real device without status.** The listener in steps 11–12 stands in for one. It is not a capture from an Xprinter, whose manual marks `DLE EOT` as valid only on serial models.
- **The register app was not launched.** 1.7.10 adds no UI and no IPC command. Its first callers are 1.7.6's status poll and 1.8.3's print path.

**Observed while validating, and reported rather than re-run away**
- **A first pass of this script at this same head** had nextest report `11 passed (1 leaky)` in the `Done when`, and `(1 leaky)` again in the first mutant's run. nextest calls a test leaky when its output handles stay open after it exits; under the repository's default nextest settings that is still a pass. In the `Done when` it was the one test without a `PASS` line, `a_profile_for_another_transport_is_refused`, which opens no socket and starts no thread. The pass stopped there because my script expected the exact summary line, and the script now records `LEAK` lines instead of filtering them out.
- **I could not make it happen again.** 18 further runs of `net::` at this head showed no leak: 10 alone, and 8 with all eight cores held at full load. This run, which the screenshots come from, showed none either. I have not shown its cause, so I'm reporting it as a rare harness-timing report, not a defect in the driver.
