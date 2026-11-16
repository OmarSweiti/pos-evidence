# 1.11.3b, part two: the component foundation on Radix, adapted to the Ledger — validation at 7e271b0

**Validated commit:** `7e271b0`, this pull request's head, in a checkout with no change to any tracked file.
**Runtime:**
- **The components:** an evidence-only page, outside the repository, imports `packages/ui`'s own Button, Dialog and direction provider from this head's source. Vite served it on 127.0.0.1 with the repository's own Vite, React and Tailwind.
- **The terminal:** its web build from `just pr`'s gate at this head, served by `vite preview` on 127.0.0.1.
- **The browser:** Chrome, driven by the pinned `@playwright/cli` 0.1.22 with `--browser chrome`, so no browser was downloaded. Reduced motion was emulated over the DevTools protocol.

**Data:** synthetic only: invented words, and one invented product and amount in the dialog.
**Emulators:** none; no device is touched by this change.

## Commands

- `pnpm --filter @pos/ui exec vitest run` at this head.
- Twelve planted defects. Each is one wrong edit to a component, `cn` or `components.json`, run on the committed tree and reverted with `git checkout`.
- `just pr`'s gate (`just pre-push`) at this head, quoted from its log; CI's required checks at this head.
- **The shadcn CLI**, as the owner decided: `pnpm dlx shadcn@4.23.0 add button dialog --cwd <scratch copy> --dry-run --view`, then read. Its output is quoted against the adaptation in the PR body.
- The 33 added packages, each looked up on the npm registry for its publish date and licence.

## Steps

| # | Step | Expected | Observed |
|---|---|---|---|
| 1 | Tests and gate (01) | 15 pass; the gate green | 15 passed. Gate: 1134 workspace tests passed (4 skipped); vitest ui 15, money 10, backoffice 3, terminal 66; both web builds, every guard suite, the secret scan. CI: rust, printer-conformance, guards, web, supply-chain and topology passed. `protected-paths` is red by design for `js-license-policy.json` alone, the 0BSD line for the owner to read. |
| 2 | Planted defects (02) | Each turns red the test aimed at it | 12 of 12 caught by the test aimed at it. P8 to P10 put back exactly what the CLI wrote: its English "Close", its black scrim, its semibold title. |
| 3 | Measured in Chrome (03) | ≥ 48 px; pressed and disabled on tokens; the ring; the dialog's direction, position, focus and motion | Every button is at least 48 × 48; control 56, target and icon 48. Pressed, sampled from the pixels: card → accent, navy → ink, tint → solid danger. After a Tab, the ring is 2 px `--color-ring`. The dialog sets `dir` on itself (`rtl` and `ltr`), renders outside `#root`, and is centred exactly. Its close button is 48 × 48 and labelled by the caller. It opens with focus on the safe action. It fades in over 200 ms ease-out, 0 s under reduced motion. Its surfaces are `--color-overlay`, `--color-popover`, `--radius-lg` and `--shadow-float`. The console shows no error from the page. |
| 4 | Supply chain (04) | Every added package at least seven days old, under a reviewed licence | 33 packages, the newest `lucide-react` 1.70.0 of 6 November. 27 MIT, 4 Apache-2.0, 1 ISC, 1 0BSD (`tslib`, through Radix's scroll lock). The shadcn CLI's 223 packages are not in the lock. |
| 5 | Buttons (05–07) | Every variant, size and disabled state, Arabic and English; the focus ring | As expected. Solid navy appears on the primary action alone, the danger word on its tint, and 3:1 outline boundaries on the icon buttons. The layout mirrors in Arabic. |
| 6 | Dialog (08–10) | Arabic and English at 1024×768; Arabic at 1920×1080 under 125 % (a 1536×864 viewport) | The close button sits at the inline end and the footer reads in each language's order. The title leaves room for the close button, and the amount is isolated left to right. |
| 7 | The terminal on the Button (11–12) | The smoke panel's toggle and action are the Button, in both directions | The toggle is `secondary`/`target` at 48 px and "Split via Rust" `primary`/`control` at 56 px. The toggle switches `dir` to `ltr`. |

## Not validated here, and why

- **WebKit and WebKitGTK** (Tauri on macOS and Linux): only Chrome rendered these. Playwright's WebKit is a separate browser download, not made for this.
- **The inventory's own components** (Numpad, PinPad, ApprovalModal and the rest) arrive with their screens, from 1.11.4 on.
- **Touch gestures** on a real panel; the pressed state was driven by a held mouse button.
