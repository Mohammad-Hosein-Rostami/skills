---
name: playwright-figma-ui-diff
description: Headed App UI vs Figma reference screenshot workflow — crop to Figma frame, evidence JSON, local pixel diff, ignore chrome/content noise. Use for Diffchecker-like pixel parity.
---

# Playwright UI Diff (App vs Figma)

“Diffchecker-like” workflow for this repo:

- **Headed Playwright MCP** (browser visible).
- **Evidence JSON** (boxes + styles + values) before trusting pixels.
- **Frame-aligned crops** — never overlay full portal chrome on a content-only Figma frame.
- **Local pixel diff** via `apps/e2e/agent-browser/scripts/figma-frame-pixel-diff.py`.

Companion: **`playwright-browser-ui-diff-data-collection`** · **`playwright-verify-fix-loop`** · **`figma-pixel-parity-check`** · **`figma-implicit-spacing`**.

## When to use

- User asks for UI exactness / Diffchecker / pixel-level Figma parity.
- After implementing a Figma screen — verify in browser and diagnose mismatches.
- Re-diff after a fix pass.

## Hard rules (false-positive killers)

1. **Same region only** — Figma `Main frame` W×H (e.g. 1180×826) vs app crop of that page shell (`data-test="…-page"` or card). Do **not** Diffchecker a full 1530×920 portal screenshot (sidebars + top nav) against a content-only Figma export.
2. **Detect white-card bounds before crop** — MCP/Figma PNGs often include **asymmetric shadow padding** (e.g. 401×560 export with card at `(20,16)–(381,536)` = 361×520, **not** a uniform 20px inset). Playwright element screenshots often include **1px box-shadow** (`362×521` → crop `(1,1)–(362,521)`). Wrong crop shifts the whole frame by 1–4px and paints **all text magenta** even when geometry is perfect.
3. **Normalize content before pixels** — sample names, emails, dates, phone formatting, live QR modules, and avatars make magenta noise. Either:
   - seed identical fixture values, or
   - mask dynamic fields in both crops (QR + avatar overhang), or
   - treat content Δ as **advisory** and gate on geometry evidence.
4. **Geometry first** — if row `y` drifts by ≥4px, fix field height / gaps before staring at Diffchecker ghosting. Cumulative `fieldH` Δ (e.g. DS medium **44** vs Figma **48**) shifts every row and the footer.
5. **Anti-alias / font** — subpixel AA and font rasterization will never be 0% pixelmatch. Pass with a ratio threshold (default **2–5%**) after crops align; use evidence JSON for spacing/typography. Structural-masked ratio (exclude QR/avatar) is the real gate.

## Inputs

1. Figma `fileKey` + `nodeId` (or URL with `node-id`).
2. App URL under `/portal/...`.
3. Viewport (default **1530×920**, DSF 1) — for capture comfort; **crop** to Figma frame for compare.
4. Optional auth / session seed.

## Mandatory loop (headed MCP)

### 1 — Figma baseline

- `get_metadata` → derive gaps (`figma-implicit-spacing`).
- `get_screenshot(fileKey, nodeId)` → `temp/figma-ui-diff/{slug}/figma-baseline.png`.

### 2 — App capture (headed)

- Navigate + `browser_resize` 1530×920.
- Seed session/fixtures so field **values** match Figma when comparing pixels.
- Screenshot full viewport → `app-actual.png` (for human context).
- Screenshot **element crop** of the Figma region:
  - prefer `[data-test="…-page"]` or the card + page header stack
  - save → `app-frame.png` (must match Figma frame aspect/size as closely as possible)

### 3 — Evidence JSON

Run **`playwright-browser-ui-diff-data-collection`**. Minimum checks — use **card-relative** coords (`el.top − card.top`):

| Check                    | How                                                                                      |
| ------------------------ | ---------------------------------------------------------------------------------------- |
| Card / frame size        | bbox vs metadata                                                                         |
| Header / footer height   | bbox                                                                                     |
| **First field abs Y**    | Must equal `content.y + firstRow.y` (often ≠ `headerH + content.pt` when layers overlap) |
| Field height / width     | `.MuiOutlinedInput-root` vs Figma input                                                  |
| Row gap                  | `row[n+1].y − row[n].y − fieldH`                                                         |
| Icon→title / button gaps | sibling bbox delta                                                                       |
| Button abs X/Y in card   | Figma actions-container x/y                                                              |
| Title typography         | computed font vs Figma text style                                                        |
| Header glyph W×H         | Must match Figma native size — fail if `DsIcon size={24}` stretched a non-square path    |

**Fail gate:** any field row abs Y Δ ≥ 2px vs metadata — fix before Diffchecker.

### 3b — Chrome / shell / label gates (card tables miss these)

Card-relative field tables + full-frame pixel ratio can **Pass** while chrome is wrong. Always assert when the Figma frame includes them:

| Check                                     | How                                                                                                       | Known miss                                                                            |
| ----------------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| **Header → first card gap**               | `card.top − back.bottom` (or `PageHeader` outer bottom) vs metadata                                       | Empty `PageHeader` stack + `spacing={4}` → **40** vs Figma **24**                     |
| **Sidebar list top inset**                | `firstMenuItem.top − navUl.top` (or nav content pad) vs Figma list `y`                                    | `paddingBlockStart:16` **+** first item `marginBlockStart:16` → **32** vs **16**      |
| **Primary CTA / split-button label wrap** | `getComputedStyle(btn).whiteSpace === 'nowrap'` **and** `scrollHeight ≤ clientHeight` (or line count ≤ 1) | `DsDropDownGroupButton` small forced `body1` → "Change Image" two lines inside 172×40 |
| **Contained dropdown typography**         | small size = **caption** (theme `sizeSmall`), not body1                                                   | Same as wrap miss                                                                     |

Do **not** call Pass on pixel ratio alone when any chrome gate fails.

### 4 — Local pixel diff

```bash
python3 apps/e2e/agent-browser/scripts/figma-frame-pixel-diff.py \
  --figma temp/figma-ui-diff/{slug}/figma-baseline.png \
  --app   temp/figma-ui-diff/{slug}/app-frame.png \
  --out   temp/figma-ui-diff/{slug}/diff.png \
  --json-out temp/figma-ui-diff/{slug}/diff-report.json \
  --threshold 32 \
  --fail-ratio 0.02
```

Show `figma-baseline.png`, `app-frame.png`, and `diff.png` in chat.

### 5 — Optional Diffchecker

Only after crops are frame-aligned. Tell the user: paste **`figma-baseline.png`** and **`app-frame.png`**, not full-chrome captures.

### 6 — Fix order

1. Layout geometry (heights, gaps, padding) from metadata.
2. DS size/variant (prefer props; zone `sx` height lock only when Figma size ≠ DS medium).
3. Typography via `DsTypography` variant.
4. Icon assets / ANCHOR missing DS icons.
5. Re-run steps 2–4.

## Output format

```markdown
## UI Diff Report — {slug}

Result: Pass | Partial | Fail

Crops: figma-baseline.png · app-frame.png · diff.png
Pixel: diffRatio=… (threshold …, fail-ratio …)

Evidence (top issues):

1. …
   Noise discounted: chrome overlay | content values | AA
```

## Mistakes

- Diffchecker full portal vs Figma content frame → fake global ghosting.
- Ignoring cumulative field-height drift (44 vs 48).
- Calling pixel Pass while geometry table still has ≥4px gaps.
- Changing shared DS defaults without `shared-component-guard` + user approval.
- Headless-only when user asked to watch the browser.
- Ad-hoc `capture.mjs` using Playwright’s default `getByTestId` (**`data-testid`**) — ICE uses **`data-test`**; prefer `locator('[data-test="…"]')` outside `agent-browser` config.
- My Memberships (`10992:327655` / `11300:293521+`): gate card **1222×710** + eco **378×180** at card-rel **y=98**; School micro **y=334** / role **y=494**. Content-height cards fail Figma whitespace — set `minHeight: 710`.
- Measuring `headerToCardGap` from title bottom instead of **Back** bottom → fake Δ3 (title sits slightly lower than the 44px Back row).
- Membership React Flow edges: **do not invent** straight/`smoothstep` edges. Download Figma Body SVG (`download_assets` `defaultFormat: svg`) and overlay the exact `Line *` / `Arrow *` paths (filled elbow+dot + stroke segments) in Body `1222×710` viewBox — see `MembershipConnectors` + `assets/connectors/*.svg`.
- Membership eco/role glyphs: **do not reuse** shared `School` / `InnoEcosystem` / `Nomad` lookalikes. `download_assets` the Figma groups (`26617:434435` School, `11305:282777` Innovation, `26626:434515` Nomad, `11300:294516` role person) and ship under `memberShip/assets/icons/`. Micro logos: use Figma image fills / node PNG export (`hollins` / `geic` / `henry_royce`).
