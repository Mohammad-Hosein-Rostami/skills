---
name: figma-playwright-parity
description: End-to-end Figma design to verified UI — metadata spacing, DS mapping, Playwright measure-fix loop. Use for implement-from-Figma plus runtime proof in one session.
---

# Figma → Playwright Parity (full chain)

Single orchestration skill for **design → code → browser proof → fix**.

## When

- Implement a Figma screen and prove it in the browser in one session
- User says “match Figma and test it” / “pixel perfect and verify”
- Regression on a Figma-driven page after layout changes

## Load order (mandatory)

| Step | Skill                                                             | Purpose                                                 |
| ---- | ----------------------------------------------------------------- | ------------------------------------------------------- |
| 1    | **`shared-library-discovery`**                                    | Reuse DS / shared-ui first                              |
| 2    | **`figma-design-to-code`** (plugin)                               | Before `get_design_context`                             |
| 3    | **`figma-design-system-parity`**                                  | Token / variant tables                                  |
| 4    | **`figma-implicit-spacing`**                                      | Gaps MCP omits or fakes                                 |
| 5    | **`figma-card-relative-geometry`**                                | Abs X/Y inside card; header/content overlap             |
| 6    | **`figma-parity-fast-path`**                                      | Geometry gate + strokes presence/absence + cropped diff |
| 7    | **`figma-page-shell-layout`**                                     | Back/breadcrumb pages, fixed cards                      |
| 7b   | **`figma-responsive-ownership`**                                  | Desktop-only Figma → full responsive matrix             |
| 7c   | **`figma-exact-not-similar`**                                     | Exact Figma — no peer substitution / micro-details      |
| 7d   | **`figma-absolute-layout-safety`**                                | Absolute/fixed overhangs across breakpoints             |
| 8    | **`figma-structure-before-sx`**                                   | DOM order ≠ Figma layers                                |
| 9    | **`figma-interactive-state`**                                     | enabled vs disabled from Figma                          |
| 10   | **`missing-ds-icon-anchors`**                                     | Icons not in IceIconsEnum                               |
| 11   | Implement in owning zone/feature layer                            |
| 12   | **`playwright-verify-fix-loop`** + **`playwright-figma-ui-diff`** | Runtime measure + frame crops                           |
| 13   | **`eslint-and-verification`**                                     | Lint + type-check                                       |

Optional: **`figma-pixel-parity-check`** for formal sign-off tables + `temp/figma-parity/` PNG pairs. Rule: **`figma-parity-geometry-gates`**.

## Phase A — Figma truth

1. **`get_metadata(fileKey, nodeId)`** — full child tree with `x,y,w,h`
2. Also open **header + footer** child nodes — record stroke **presence or absence** (do not invent footer dividers)
3. **`figma-implicit-spacing`** + **`figma-card-relative-geometry`** — derive gaps; build abs Y table
4. **`get_design_context`** on **sub-nodes** if payload truncates — never paste verbatim
5. **`get_screenshot`** — visual reference @1x; only add strokes with positive Figma evidence
6. Build DS mapping tables (`figma-design-system-parity`)

## Phase B — Implement

- DS components + `sx` + theme tokens only
- Spacing from **derived metadata**, not guessed `Stack spacing`
- i18n via `useTranslation` — no hardcoded copy
- `shared-component-guard` before editing `libraries/shared/*`

## Phase C — Playwright proof

Load **`playwright-verify-fix-loop`**:

1. Route under `/portal/...`
2. Viewport = Figma / agent desktop (**1530×920** default) for geometry + cropped pixel diff
3. Snapshot → screenshot → **boxes: true** → evaluate key nodes
4. Compare table: Figma derived vs runtime
5. Fix failures at owning layer; **max 3 rounds**
6. **Responsive ownership check** (even if Figma has only one frame): load **`figma-responsive-ownership`** — phone + tablet + oversized matrix (not only 1530×920); edge-anchored CTAs; fluid cards; no horizontal scroll. Companion: **`figma-page-shell-layout`**.
7. Attach PNGs from `temp/agent-browser/` in chat (1530 + 375 + 768 + 1920)

## Phase D — Sign-off

| Gate             | Tool                                      |
| ---------------- | ----------------------------------------- |
| Structure + copy | Playwright snapshot roles/text            |
| Spacing          | Metadata math + Playwright boxes          |
| Edge CTAs        | Desktop abs + larger-viewport inset check |
| Colors/type      | DS tokens + optional computed style read  |
| Visual           | Screenshot side-by-side with Figma export |
| Static           | ESLint + type-check                       |

## Spacing failure class (common)

```
MCP says gap: 18px → Dev Mode shows none → derive from metadata → 24px → fix Stack gap → Playwright confirms 24px
```

Always document when MCP gap was **rejected**.

## Edge-pin failure class (common)

```
Figma Save & Next at y=758 in 826-tall Main → agent uses mt:28 under card → Pass at 1530×920 → Fail on tall pane (button mid-page)
→ Detect right/bottom inset ≈ 24 → contentHeightFixed fill + flex-end footer → recheck desktop + oversized
```

## Out of scope

- GraphQL / new REST without OpenAPI contract
- Mass E2E unless user asks
- Production credentials in MCP
- Storybook

## Companion skills

- `fullstack-feature-delivery-loop`

## Docs

- `docs/figma-ui-verification.md`
- `docs/agent-browser.md`
