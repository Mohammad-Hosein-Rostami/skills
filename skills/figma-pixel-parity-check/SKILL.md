---
name: figma-pixel-parity-check
description: Pixel perfect, match Figma, visual diff, compare design, figma parity. Use when user provides a Figma URL or frame and a running app route, component path, or screenshot compare request.
---

# Figma Pixel-Parity Check

**Primary skill** for aligning coded UI to Figma. Guardrails: `skill-first-governance` + Figma companion skills · Reference: `docs/figma-ui-verification.md`.

**Always load `figma-design-system-parity` in the same session** — token/variant mapping happens before code edits.

**Spacing MCP gaps:** load **`figma-implicit-spacing`** when `get_design_context` gap/padding disagrees with Dev Mode or sibling coordinates. Runtime proof: **`playwright-verify-fix-loop`**. Full chain: **`figma-playwright-parity`**.

Portal dev: `http://localhost:4200/portal/...` (`apps/portal/vite.config.ts`).

## Inputs required from user

- **Figma:** file URL + `node-id` or frame name
- **App:** URL or route + viewport W×H (e.g. 1440×900)
- **Optional:** theme mode, locale, auth note, open drawer/modal, component path

Stop if Figma URL or viewport is missing.

## Prerequisites

- Load **`figma-design-system-parity`** — DS token/variant mapping (mandatory)
- Load **`shared-component-guard`** — before any `libraries/shared/*` edit
- Load **`api-contract-first`** — for ICE Calls / REST parity audits
- Load **`figma-use`** before every `use_figma` call (Figma plugin skill)
- **`design-system-and-styling.mdc`** + **`shared-library-discovery`**
- `pages/iceCalls/**` → **`ice-calls-jitsi`**
- Figma MCP: `mcp_auth` for `plugin-figma-figma` if needed

Code → Figma: **`figma-generate-design`**.

Spacing truth for this repo: `theme.spacing(1) = 4px` (`spacing.ts`), not default MUI `8px`.

## Workflow

1. **Figma baseline** — `get_metadata` first (x/y/w/h for every child) → **`figma-implicit-spacing`** derive gaps/padding → `get_design_context` on sub-nodes: fills, text styles, variants; export PNG @1x. Build mapping tables per `figma-design-system-parity`. **Do not trust MCP `gap` when Dev Mode omits it.**

2. **DS alignment pass** — For each Figma value, resolve to `DsTypography` variant, `palette.*`, `spacing(n)` (4px scale), DS component variant. List gaps: extend DS vs fix instance.

3. **Separate design truth from sample dressing** — mark which Figma values are authoritative (spacing, sizes, px-per-unit scale, tokens, structure, variants) and which are placeholder content (sample text, names, dates/times, and the canvas position of sample blocks). Placeholder content **never** becomes fixture data. If a mock's block position contradicts its own label text, resolve to one real value, derive the rest, and report the inconsistency. Rule: `derived-display-values.mdc`; skill: **`calendar-time-grid-accuracy`** for anything where position/size encodes a quantity.

4. **Clarification gate** — if layout/spec is ambiguous, unusual, or conflicts with DS defaults, ask the user before implementing structural/shared changes.

5. **App capture** — Playwright MCP: `browser_navigate` → viewport = Figma size, DSF 1 → auth → `browser_snapshot` **`boxes: true`** + `browser_take_screenshot` (`scale: css`) → save under `temp/agent-browser/` and show user. Skill: **`playwright-verify-fix-loop`**.

6. **Structured diff** — table: Property | Figma | Code | Token/variant target | Δ | Fix file

7. **Visual diff** — save to `temp/figma-parity/{slug}/figma.png` + `app.png`; vision summary. For Diffchecker / local pixels load **`playwright-figma-ui-diff`**: crop app to the **Figma frame** (not full portal chrome), normalize sample content, then run `apps/e2e/agent-browser/scripts/figma-frame-pixel-diff.py`. Optional [Diffchecker](https://www.diffchecker.com/image-compare/) only on those aligned crops.

8. **Fix pass** (ordered):
   1. Extend `designSystem` / `shared-ui` if gap is reusable
   2. DS component props (variant, size, color)
   3. `DsTypography` variant (not raw fontSize)
   4. `palette.*` / `color="iceGray.*"` (not hex)
   5. `sx` spacing via theme
   6. Last-resort px + comment

   i18n via `@ice-web-app/shared-hooks`, not hardcoded copy.

9. **Re-verify** — repeat diff/capture loop until within tolerance or blocked.

10. **Verify** — ESLint/Prettier on touched files; `pnpm nx run <project>:type-check`. No new tests unless asked. Storybook is removed.

## Tolerance

| Exact (fix)                                             | Acceptable (document)                 |
| ------------------------------------------------------- | ------------------------------------- |
| Wrong DS component or variant                           | 1–2px subpixel                        |
| Custom hex/fontSize when token exists                   | Font rendering / AA                   |
| ≥4px spacing                                            | Scrollbars                            |
| Wrong typography variant                                | Live data vs placeholder              |
| ≥2px radius                                             | Long i18n strings                     |
| Wrong layout                                            | ICE Calls video/canvas                |
| Field height Δ that accumulates (≥4px per row vs Figma) | Documented DS size lock on one screen |

**Cumulative height:** if Figma inputs are **48** and DS `medium` paints **~44**, every row drifts and footer/buttons ghost in overlays — fix with zone-local height lock (or approved DS change), not by tweaking random margins.

## Output format

Parity report with **Pass / Partial / Fail** per section; **DS mapping tables**; spec + visual summary; files changed + token mappings; `Extend DS` vs `instance fix` breakdown; remaining manual checks.

## Mistakes

Skip `figma-design-system-parity` · skip `figma-use` · **trusting MCP `gap` without metadata coordinate check** · **skipping `get_metadata` before `get_design_context` on large frames** · viewport mismatch · wrong spacing scale (`8px` assumption) · hex/Tailwind · fork DS components · raw MUI Dialog · API/store/i18n changes · ICE Calls PNG on video · reintroducing Storybook · paste Figma MCP code verbatim · shared default change without approval · **treating a sample block's canvas position as a real value** · **storing a mock's placeholder label next to the data that produces it** · **reproducing a mock's internal contradiction instead of resolving and reporting it** · **header-only 24px when body shares the same frame `x`** · **inventing page `px`/`py` on top of portal padding** · **using fluid `Grid md={4}` for fixed-width Figma cards** · **locking `minWidth` to desktop card width on mobile (right padding disappears)** · **desktop-only header→body gaps with no responsive taper** · **`Stack spacing={6}` for a non-token header→body gap** · **not re-checking `cards.y − header.bottom` after padding changes** · **rendering `"undefined undefined"` names while loading** · **parity only at desktop — skip phone viewport when page has `p: 24` + fixed cards**

Page shells / Back + breadcrumbs / fixed card rows → load **`figma-page-shell-layout`**. On phone: confirm padding on **both** sides and that header→body gap is smaller than desktop.
