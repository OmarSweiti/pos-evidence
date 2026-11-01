# Validation: microstep 1.6.8, PII scrubbing in the log layer

**Commit validated:** `96122d1`, the pull request's head, checked out clean in a scratch worktree.

**How it ran:**
- **No screen to drive.** The Phase-0 smoke panel's one command, `split_tender`, doesn't render its errors, and the sale commands that will log through the layer are 1.4.11's. So the evidence is the PR's own code in a release build:
  - `SENSITIVE_FIELD_RULES`;
  - `ScrubbingLayer`, `scrub_json` and `telemetry::init`;
  - `check_payload` and `AuditRepository::append`, on a register database opened by `pos_db::open`;
  - `IpcError`.
- **A throwaway harness**, not in the repository, built offline in release against the repository's own `Cargo.lock`. Every name, phone, email, card number, PIN and secret is synthetic. `init` runs in a child process, as `run()` calls it, and its real standard error is captured.
- **Planted defects:** each one changed one place in the worktree, ran that crate's tests for this step, and was restored. The run ends with "restored: 0 changed files", and the harness was built after it.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | `cargo nextest run -p terminal telemetry::tests:: && cargo nextest run -p terminal ipc_errors_` (the `Done when`) | every test passes | **pass**: 5/5 and 2/2 (01) |
| 2 | The registry's and the audit guard's own tests in `pos-domain` | every test passes | **pass**: 3/3 (01) |
| 3 | The terminal crate, `pos-domain`, the workspace | no failure | **pass**: 34/34, 360/360, and 787/787 with the 4 existing `#[ignore]` tests skipped (01) |
| 4 | The registry, and eighteen field names in the spellings a producer might use | every listed name in any spelling redacted; innocent names such as `masked_pan`, `pinned` and `hashtag` kept | **pass** (02) |
| 5 | A sale's trace through the layer: a named buyer, a phone, an email, a card number, a manager's PIN and a provider's secret, logged as fields, by the cart's `Debug`, as a nested JSON receipt, as JSON held in text, and inside a span carrying a phone | none of the six values in the trace; every line one JSON object; the sale's id, TIN, masked PAN and scheme kept; the span named, its field not written | **pass**: 11 values redacted (02) |
| 6 | `scrub_json` on a payload nested four deep, with JSON inside a string | nothing listed survives | **pass** (02) |
| 7 | `telemetry::init` in its own process, logging one `INFO` line with a phone and one `DEBUG` line with an email | one JSON line on standard error with the phone redacted, and no `DEBUG` line | **pass** (02) |
| 8 | `check_payload` on two clean payloads and five that name a listed field, nested up to four deep or camel-cased | the clean ones accepted; each other refused at the member's place, never its value; with two faults, the first in canonical order is reported | **pass** (03) |
| 9 | `AuditRepository::append` on a register database, with a phone in the payload, then without | the first refused before anything is written, with `audit_log` still empty; the second appended at seq 1 | **pass** (03) |
| 10 | `split_tender`'s failure through `IpcError::from_source`, and a database refusal naming its table and column | the webview gets a code, a key, a static detail and a trace id, never the source text; the log has the source under the same trace id | **pass** (03) |
| 11 | Five failures' trace ids | all distinct, all random (version 4) | **pass** (03) |
| 12 | Twenty-six planted defects | each caught | **pass** (04): all twenty-six caught, worktree restored |

## Notes

- **The planted defects found two gaps, both closed before the pull request opened.**
  - Nothing tested `init` itself: a defect that installed nothing, or logged at `DEBUG`, passed. `init_installs_the_layer_as_the_global_default` now catches both.
  - No property covered the audit guard. `prop_a_never_log_name_is_refused_wherever_it_nests` was added, and passed 65,536 cases.
- **`split_tender` is private to the shell,** so step 10 reproduces its body: `split_evenly`, mapped through `IpcError::from_source`.
- **The redaction itself is visible on board 02,** as `[redacted]` in each line. The checks name each synthetic value by its first eight characters.

## Not validated, and why

- **A launched register window:** the smoke panel shows no error, and no command that logs a customer exists before 1.4.11.
- **`no_pii_in_a_captured_panic`:** 3.9.1's, with crash reporting.
- **The diagnostic bundle's scrub:** 1.11.9's.
