# 1.11.3b, part one: the register's look chosen, its tokens held to DESIGN.md — validation at f7544a3

**Validated commit:** `f7544a3`, this pull request's head, in a checkout with no change to any tracked file.
**Runtime:** both front ends' stylesheets now import `@pos/ui/tokens.css`, with Tailwind's own palette removed. Each app's web build from `just pr`'s gate at this head was served by `vite preview` on 127.0.0.1 and captured in headless Chrome. The sale-screen comps are static HTML that read the repository's `packages/ui/src/tokens.css` at render time, so they cannot drift from it. They were captured at 1024×768, and at 1920×1080 as Windows draws it at 125 % (a 1536×864 viewport at device scale 1.25).
**Data:** synthetic only: an invented store, invented staff names and products. Every figure is typed into the comp. The tax figures are illustrative: 16 % included on four of the products, nothing on the rest. They are not the tax engine's output.
**Emulators:** none; no device is touched by this change.

## Commands

- `pnpm --filter @pos/ui exec vitest run src/tokens.test.ts` (the step's `Done when`), at this head.
- Eight planted defects, each one wrong edit to `DESIGN.md` or `tokens.css` on the committed tree, run, then reverted with `git checkout`.
- `just pr`'s gate (`just pre-push`) at this head, quoted from its log; CI's required checks at this head.
- impeccable's `critique` and `audit`, run by two reviewers working apart, on the comps as drawn before this pull request's fixes. The audit measured the rendered DOM over the DevTools protocol, because impeccable's detector is deliberately not installed (`ref/ui-design.md` §6). The auditor's own tools were then re-run on the redrawn comps.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | Tests and gate (01) | 3 token tests pass; the gate green | 3 passed. Gate: 1134 workspace tests passed (4 skipped), vitest ui 3, money 10, backoffice 3, terminal 66, both web builds, every guard suite, the secret scan. Every required CI check passed at this head but `validation`. |
| 2 | Planted defects (02) | Each turns a test red | 8 of 8 caught. Each named the test meant for it: a value or token out of lockstep reds `design_tokens_match_design_md`; a pair below its floor, or TOTAL's 7:1 floor relaxed, reds the contrast test. |
| 3 | Critique and audit, answered (03) | Every P0 and P1 answered | No P0. One P1, found by both reviews: the selected cart line, the one `Del` and `+`/`−` act on, was marked only by a 1.14:1 tint. Now two 2 px navy rules mark it (9.04:1 on the tint), and only that line carries the quantity controls and line menu. Every P2 is answered and re-measured; every P3 is answered or carried (board 03). Critique 24 of 36, audit 15 of 20, before the fixes. |
| 4 | The Ledger, redrawn (04–07) | Arabic and English at 1024×768 and 1920×1080 at 125 %; TOTAL ≥ 7:1 and the largest element | TOTAL 17.28:1, 48 px at 1024 and 64 CSS px (80 device px) at 1920, closed beneath by the double rule. Every text pair ≥ 5.42:1. Every control ≥ 48 px. Amounts align on their decimal point, a spread of 0 px. No overflow, no clipped text, and both embedded weights loaded with no fallback face. In English at 1024 the favourites' third row gives way whole because "Dish soap 1 L" wraps. |
| 5 | Edge states (08–09) | The offline promise, a printer badge, the training banner, a chosen category, the focus ring, an earlier-lines row, in both languages | As expected. The badges are buttons the strip's height (48 px), the earlier-lines row is 48 px, and in Arabic `13.500` and `0.800` share one decimal point. Redrawing found a real overflow: the English offline copy widened the screen. Fixed: the screen never widens, and the strip drops the shift's label first. Measured clean at 1024×768 and 1024×640, the declared minimum window. |
| 6 | Before the review (10) | The comp the critique and audit read | The faint selected line, the rule above the label, the solid-navy chip. |
| 7 | The three candidates (11–13) | What the owner chose from on 16 November 2026 | The Ledger (chosen), the Signboard and the Receipt, as shown. |
| 8 | The apps on the tokens (14–18) | The terminal in Arabic and English, its warning state and focus ring; the back office | The terminal draws in the embedded face, on the white ground and navy ink, with 3:1 field borders, its one action in solid navy, the warning tokens on its refusal, and the 2 px ring after a Tab. The back office shows the muted ground and ink in the system face, because `DESIGN.md` has it load the embedded face with its first screen. |

## Not validated here, and why

- **The shadcn/ui foundation** is this step's second part; no component exists yet.
- **The sale screen's behaviour** (scan to line, the numpad, the line menu, the void) is 1.11.5's. The comps are design records.
- **Glare on a real panel and WebKit's rendering** (Tauri on macOS, WebKitGTK on Linux): only Blink rendered these; the panels are 5.8's hardware qualification.
- **The Arabic screen's currency label** (`JOD`, `د.أ` or `دينار`) is open for the owner, to match the receipt, whose label is not decided.
- **The Phase-0 smoke panel's copy** is English placeholder text from before the catalogue. In right-to-left its full stop sits at the start of the refusal line. 1.11.5 replaces the panel.
