---
name: figma-design-system-parity
description: Design-system-first Figma parity — map Figma variants, colors, typography, spacing, and radius to @ice-web-app/designSystem tokens before touching feature code. Use with figma-pixel-parity-check for any pixel-perfect or implement-from-Figma task.
---

# Figma Design-System Parity

**Companion skill** to `figma-pixel-parity-check`. Use this **before** applying visual fixes in zone/ecosystem code.

Guardrails: `skill-first-governance` + `shared-component-guard` + `shared-library-discovery`.

Before editing `libraries/shared/*`, load **`shared-component-guard`**.

## Principle

**Shared-first, token-first, variant-first.** Figma values must land on existing DS primitives and theme tokens. Extend shared packages when a gap is real and reusable — never fork one-off styling into a feature folder.

## Mandatory discovery order

1. `@ice-web-app/designSystem` exports and component props (variant, size, color)
2. `@ice-web-app/shared-ui` composites already used in the feature area
3. Theme tokens:
   - `libraries/shared/designSystem/src/materio/@core/theme/colorSchemes.ts`
   - `libraries/shared/designSystem/src/materio/@core/theme/typography.ts`
   - `libraries/shared/designSystem/src/materio/@core/theme/breakpoints.ts`
4. Real usage in the target feature folder (match local conventions)
5. Figma MCP spec (`get_design_context` / `get_metadata`) — last as **source of truth for deltas**, not as code to paste

## Figma → code mapping tables (fill per frame)

### Typography

| Figma text style | Size | Weight | Line height | `DsTypography` variant        |
| ---------------- | ---- | ------ | ----------- | ----------------------------- |
| (extract)        |      |        |             | `body2`, `body2_500`, `h5`, … |

Rules:

- **Never** set `fontSize`, `fontWeight`, `lineHeight`, or `letterSpacing` in `sx` when a variant matches within 1px.
- **Never** use `sx={{ typography: 'caption' }}` (or any theme key) to fake text styles — render `DsTypography variant="…"`.
- Button/group labels that need Figma text styles: wrap with `DsTypography variant="…" component="span"`.
- If no variant matches and the style appears in **≥2 places** or is a DS primitive concern → add variant to `typography.ts` + export via `DsTypography`, then use it.
- One-off 1–2px drift: prefer nearest variant; document acceptable drift — do not invent `fontSize: 13px`.
- Skill: **`ds-typography-first`**.

### Color

| Figma fill/stroke | Hex (reference) | Token                  | Usage |
| ----------------- | --------------- | ---------------------- | ----- |
|                   |                 | `palette.iceGray[700]` | text  |
|                   |                 | `palette.primary.main` | CTA   |

Rules:

- **Never** hardcode hex in feature `sx` when `palette.*` or `color="iceGray.500"` exists.
- Semantic colors: `primary`, `secondary`, `error`, `success`, `warning`, `info`, `iceGray[*]`, `iceBlue[*]`, etc. from `colorSchemes.ts`.
- New recurring color → add token in `colorSchemes.ts`, not inline in zone code.
- `bgcolor: (theme) => theme.palette...` preferred over template literals with hex.

### Spacing & layout

| Figma gap/padding | px  | MUI `spacing(n)` | Notes                   |
| ----------------- | --- | ---------------- | ----------------------- |
| 4                 | 4   | `1`              | repo spacing base = 4px |
| 8                 | 8   | `2`              |                         |
| 12                | 12  | `3`              |                         |
| 16                | 16  | `4`              |                         |
| 20                | 20  | `5`              |                         |
| 24                | 24  | `6`              |                         |

Rules:

- Prefer `Stack spacing={n}`, `gap`, `p`, `px`, `py` with theme spacing — not raw `px` unless Figma requires non-token values.
- **When MCP `gap`/padding is missing, computed, or contradicts Dev Mode** → load skill **`figma-implicit-spacing`**: derive from `get_metadata` sibling coordinates; verify with Playwright `boxes: true`.
- Multi-column / left-center-right regions → MUI **`Grid` `container`/`item`** (skill **`mui-layout-grid`**). Never `sx={{ display: 'grid', gridTemplateColumns }}`.
- ≥4px mismatch vs Figma → fix.
- When Figma uses auto-layout **`gap`**, map it explicitly — do not approximate with only child padding. Mini calendar example: root `gap 24` + body `gap 6` + Dates `pt 12` (skill **`figma-structure-before-sx`** / **`ice-calls-calendar-layout`**).
- **`SPACE_BETWEEN` / gap Auto** → `justifyContent: 'space-between'`, not a fake fixed `gap` from API `itemSpacing`.

### Radius, shadow, border

| Figma                                     | Token / DS prop                         |
| ----------------------------------------- | --------------------------------------- |
| 6px radius                                | `borderRadius` theme or DS card default |
| ice card shadow                           | `boxShadow="iceDefault"` on `DsCard`    |
| Perfect circle (selected day, round well) | `borderRadius: '50%'` — never `'500px'` |

Check `DsCard`, `DsButton`, `DsChip`, `DsDrawer`, `OptionMenu` defaults before overriding.

When nesting theme typography/palette under CSS selectors in `sx`, use a **top-level** `(theme) => ({ … })` — skill **`mui-sx-theme-callback`**.

### Component choice & variants

| Figma layer / component      | DS component                        | Variant / size / color                                                                             |
| ---------------------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------- |
| Outlined input 40 / 44 / 48  | `DsTextField`                       | `size="small"` / `"medium"` / `"large"` (D-0006)                                                   |
| Primary CTA                  | `DsButton`                          | `contained` `primary` `small`                                                                      |
| Tonal badge                  | `DsChip`                            | `tonal` `primary` `small`                                                                          |
| Overflow menu                | `OptionMenu`                        | reuse `iceCallsMenuPaperSx` pattern                                                                |
| Compact month (sidebar)      | `DsMiniCalendar`                    | default width 345                                                                                  |
| Inline toolbar date          | `DsDatePicker`                      | `variant="inlineText"`                                                                             |
| Time-only list (`578:96935`) | `DsDatePicker` `showTimeSelectOnly` | styles in `AppReactDatepicker` `&.react-datepicker--time-only` — not the date-calendar time column |
| ICE Calls calendar body      | two `DsCard`s                       | board + handbar, gap 16 — `ice-calls-calendar-layout`                                              |

Rules:

- Match **variant set** from Figma (contained/outlined/text, tonal/filled, circle icon button, drawer vs modal).
- Do not use raw MUI `Button`, `Dialog`, `Menu` when DS equivalent exists.
- If Figma uses a DS component instance, find its Code Connect mapping or nearest real usage in repo.

### Icons

Load **`ds-icon-svg-assets`** — it owns this whole area. The short version:

| Figma glyph | Render as                                     | Size                     |
| ----------- | --------------------------------------------- | ------------------------ |
| setting cog | `<DsIcon customIcon={<Setting />} size={20}>` | exact Figma px as number |

- The component comes from `libraries/shared/assets/icons`, imported from `@ice-web-app/assets`.
- `icon="SETTING"` (the `IceIconsEnum` webfont) is legacy. Do not add new ones.
- Pass the measured Figma px (`size={20}`), not the `sm`/`md`/`lg` tokens — those belong to the font path.
- **Given a Figma glyph, prove it does not already exist before creating one:**
  `pnpm icons:match temp/icons/<slug>.svg`. `IDENTICAL` / `RESCALED` means reuse; `NEAR` /
  `CANDIDATE` means pixel-diff before deciding. Name similarity is not evidence — `TRASH` is the
  existing `Delete`, and `INFO_CIRCLE` is **not** the existing `Info`.
- No component and no match? Ask the user for the Figma node link, then create it in
  `libraries/shared/assets/icons` per `ds-icon-svg-assets`. Never approximate the geometry.
- **Do not carry Figma's fill into the glyph.** A grey fill (`#757575`, `#424242`, `#E0E0E0`) means
  "untinted" — map it to `currentColor` and pass the token at the call site
  (`color="iceGray.500"`). Baking it makes `DsIcon color` a no-op. The single exception is a two-tone
  semantic illustration: strong + `100` pair of one tone, tone declared in a registry. Table and
  rules in `ds-icon-svg-assets/references/color-conventions.md`.
- `figma-interactive-state` covers reading the _intended_ state colour off Figma's fills rather than
  inferring it from the screenshot — use it before deciding a glyph's token.

## Extension decision tree

When Figma does not match existing DS:

```
Is it reusable outside this one line of sx?
├─ YES → extend designSystem (variant, token, prop) or shared-ui composite
└─ NO  → minimal sx on the instance using closest tokens + comment why
```

If extension requires changing shared defaults, stop and request user approval with blast-radius summary.

**ICE Calls shared patterns** (prefer extending these before new constants):

- `utils/iceCallsInCallMenus.ts` — menu paper, dividers, destructive item
- `utils/iceCallsInCallLayout.ts` — control bar dimensions
- `utils/iceCallsHubList.ts` — accordion hub sections

Add to these files when the same Figma spec applies to multiple in-call menus/drawers.

## Parity audit checklist (per frame)

Pass / Partial / Fail each:

| Area            | Checks                                               |
| --------------- | ---------------------------------------------------- |
| Typography      | variant, color prop, text transform, alignment       |
| Color           | fills, borders, icons, states (hover/disabled/error) |
| Spacing         | padding, gap, margin, section rhythm                 |
| Sizing          | button size, avatar size, drawer width, min-heights  |
| Radius & shadow | card, menu, chips, video tiles                       |
| Component       | correct DS primitive + variant                       |
| States          | default, hover, active, disabled, loading, error     |
| Responsive      | xs/sm/md frames if provided                          |
| i18n            | no hardcoded copy                                    |
| A11y            | aria-labels on icon-only controls                    |

## ICE Calls scope boundaries

- **In scope for pixel diff:** chrome, control bar, drawers, menus, tags, cards, headers, empty/loading states
- **Out of scope:** live WebRTC video pixels, waveform animation timing, dynamic participant names/avatars

Load **`ice-calls-jitsi`** for any `pages/iceCalls/**` work. Visual-only unless user asks for behavior/API changes.

## Output

1. Filled mapping tables (typography, color, spacing, components)
2. Gap list: `Extend DS` vs `Fix instance` vs `Acceptable drift`
3. Files to touch (ordered: designSystem → shared-ui → zone feature)
4. Pass/Partial/Fail summary

## Mistakes

- Pasting Figma MCP Tailwind/HTML output into zone code
- `fontSize: 14` instead of `variant="body2"`
- `#434356` instead of `iceGray.700`
- Assuming MUI default spacing scale (8px) in this repo
- Duplicating `OptionMenu` paper styles in every drawer
- New `Modal` instead of `DsDrawer`
- Adding tokens for a single arbitrary value used once
- Skipping `shared-library-discovery` before creating helpers
- **Assuming library DOM order matches Figma layer order** — e.g. MUI `PickersCalendarHeader` is `[label][arrows]` while Figma is `[prev][label][next]`. CSS on ArrowSwitcher cannot fix that; replace the slot. Skill: **`figma-structure-before-sx`**.
- Implementing nested chrome from a parent screenshot only — open the **header/control node** (`25602:477718` for mini calendar header) and read widths (218 cluster, 44 buttons).
- Calling parity done without screenshot-diffing **your build** vs Figma/UI reference.
- **Calendar Dates height math wrong** — Figma Dates is **200** (`pt + 5×36 + pb`). `180` (no padding) clips; `236` (6 weeks) breaks overall **302**.
- **Ignoring Figma flex `gap`** — ReactDatePicker root `gap: 24` and body `gap: 6` are the header↔weekday and weekday↔Dates spacing. Dates `pt` alone is not enough. Skill: **`figma-structure-before-sx`**.
- **Not re-checking root frame size** after a local fix — mini calendar must stay **`345×302`**.
- **Page shell padding guesswork** — inventing `Stack px/py={6}` or header-only `pt/pl: 24` instead of `setContentPadding(0)` + Figma **page** inset when header+body share `x`, plus desktop header→body `mt` = `cards.y − header.bottom` with **responsive taper** if no mobile frame. Skill: **`figma-page-shell-layout`**.
- **Fluid Grid for fixed Figma cards** — `md={4}` stretches 361px cards; use centered `Stack` + `maxWidth` + fluid mobile width + exact gap. Skill: **`figma-page-shell-layout`**.
- **Desktop `minWidth` on phone** — locking `minWidth: 361` (or any Figma card width) when viewport − page padding is smaller → right padding “gone”. Skill: **`figma-page-shell-layout`**.
- **Desktop-only section gaps** — only `mt: '84.5px'` with no `xs`/`sm` taper when only a desktop frame exists. Skill: **`figma-page-shell-layout`**.
- **Loading shows `"undefined"`** — interpolating `firstName`/`lastName` before data exists; keep avatar + title on skeletons until names are real.
- **Person `DsSelect` initials instead of photos** — `src: downloads/profile-picture/…` 404s on the portal; use `imgUrl` + `getImageApi`.
