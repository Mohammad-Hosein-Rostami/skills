---
name: ds-icon-svg-assets
description: Canonical icon workflow — every DsIcon must render a shared SVG component from libraries/shared/assets/icons via customIcon, never the IceIconsEnum icon font. Use when adding, refactoring, or reviewing any icon, wiring a Figma glyph into code, or continuing the icon-font migration.
---

# DsIcon + Shared SVG Icon Assets

Companion skills: `figma-design-system-parity`, `shared-library-discovery`, `ds-typography-first`,
`playwright-figma-ui-diff`, `eslint-and-verification`. Supersedes the feature-local placeholder flow
in `missing-ds-icon-anchors`.

## Tooling

| Command                                         | Purpose                                                        |
| ----------------------------------------------- | -------------------------------------------------------------- |
| `pnpm icons:match temp/icons/<slug>.svg`        | **Reuse gate.** Does this Figma glyph already exist?           |
| `node scripts/icon-match.mjs --icon A --icon B` | Compare two icons already in the set                           |
| `pnpm icons:audit`                              | Cross-compare the set — duplicates, paint modes, tone registry |
| `references/icon-catalog.md`                    | All 125 icons with dimensions, `viewBox`, paint mode           |
| `references/reuse-aliases.md`                   | Known aliases, rotation family, false friends, reserved names  |
| `references/color-conventions.md`               | **Paint modes, the tone table, and call-site colouring**       |

## Hard rules

1. **Every new or touched `DsIcon` renders a shared SVG component**, not the icon font:
   `<DsIcon customIcon={<Search />} size={20} />`.
2. `<DsIcon icon="SEARCH" />` (the `IceIconsEnum` webfont path) is **legacy**. Never add a new one.
   Migrating one is always in scope when you touch its file.
3. The SVG component lives in **`libraries/shared/assets/icons/<Name>.tsx`** — one icon per file,
   never in a feature `assets/` folder, never inline in a page.
4. It is imported **only** from the barrel: `import { Search } from '@ice-web-app/assets';`
5. Colour comes from `DsIcon`, so the SVG paints with `currentColor`. The **only** exception is a
   two-tone semantic illustration, which bakes one tone's strong + `100` pair and declares that tone
   in a registry — see `references/color-conventions.md`. A baked neutral grey is always a bug.
6. Never hand-draw a glyph. Pull the real path data from the Figma node.
7. **Never create an icon before running the reuse gate below.** Given a Figma link, prove by
   measurement that the glyph is not already in the set. A near-duplicate file is worse than no file:
   the two drift, and pixel parity silently depends on which one a call site picked.
8. **Never ship `stroke="currentColor"` without `strokeWidth` from Figma** (or a documented optical
   substitute after screenshot QA). Bare stroke defaults to `1` and shreds exploded exports.
9. **CTA icons inside `DsButton` must set `color` or `inherit !important`** — contained primary →
   `iceGray.50`; outlined → match the label (`primary.500`).
10. **No `react-icons` (or other third-party icon packs) for product chrome** — shared SVG only.
11. **Always visual-check at call-site size** after any fill+stroke conversion (`size={16}` / `20`).
12. **Linked Figma node = geometry authority. Never substitute a different product’s glyph.** If the
    user (or inventory) links Collab Zone nodes for `CallIncoming` / `CallOutgoing` /
    `CallNoAnswered`, ship **those** path groups (`Group 1010107929` / `7935` / `7941`). Do **not**
    replace them with management-info solid phones, remix icons, or any “cleaner” lookalike because
    the MCP export is exploded / hard to paint. Allowed: convert **the same paths** to
    `currentColor` (fill-only on messy fill+stroke, or explicit `strokeWidth` on stroke-only
    overlays). Forbidden: downloading a different SVG from another file and calling it fixed.
    **Direction arrows:** MCP Vector exports are shredded bars without a readable head. Reconstruct an
    **open 90° ⌞/⌝** (H + V arms + diagonal shaft) from Figma CleanShot / tip·arm bboxes — hub
    `CallOutgoing` / `CallIncoming` are open stroke corners with **round** caps/joins, **not** filled
    triangles (filled heads fail Figma parity). Arms ≈4.35 vb, stroke ≈1.1 on the 16 vb. Always verify
    at 16px **and** zoom against the linked CleanShot.
13. **Tracker status is part of the refactor — same turn as the code.** When you migrate any
    `icon="KEY"` (or finish a `customIcon` swap the inventory listed), you **must** mark that call
    site done before claiming the task complete. Leaving the row as pending / `→ manual` while the
    file already uses `customIcon` is a process bug — the next agent will re-do the work.
    - **Collab zone files:** update `docs/collab-zone-icon-refactor.md` **and**
      `docs/icon-refactor-inventory.md` (Part 3 / unresolved lists).
    - **Everywhere else:** update `docs/icon-refactor-inventory.md` at minimum.
    - Format under the file bullet: `✅ DONE — <shipped component(s) + short note>` then keep the
      `L## · \`KEY\` → \`<Component />\``lines with the **actual** import name (aliases count:`TRASH`→`<Delete />`, social URL → `<IceLink />`not`<Website />`, `WEBSITE`globe →`<Website />`).
    - Remove the file from any **Unresolved / dynamic** list once it no longer uses the font API.
    - Do **not** invent a `Trash.tsx` just because the inventory “Proposed” column says `Trash`.
14. **Call-site `size` = measured glyph box — never inflate; measure outer vs leaf.** Legacy webfont
    icons often looked optically small, so old call sites used `size="xl"` (**28**) or a 32×32
    `IconButton` around a 24 glyph. Shared SVG components already match Figma geometry — **do not**
    bump `DsIcon size` or the hit-target above the linked instance to “make it look right.”

    **Critical — `ice-Close` on card / Clicked-cards overlays** (`10379:379433`, `10270:294833`):
    - Figma **outer** instance = **24×24** (hit-target / `IconButton` with `p: 0`).
    - Figma **leaf** Vector is inset (~5px) ≈ **14×14** — paint `#424242` → `color="iceGray.700"`.
    - Call site: `size={14}` (same as `DsModal` header close). **Never `size={24}`** on the glyph —
      that stretches the 16×16 `Close` component to fill the outer frame and looks oversized/heavy
      (and often “too black”). **Never `iceGray.500`** for this chrome — that is secondary text grey.
    - `ActionButton` must not default to **28** (old webfont inflation). Default lg glyph is **24**;
      card-modal close passes `iconSize={14}` + `customIcon={<Close />}` + `color="iceGray.700"`.
    - **Hit-target width trap:** theme `MuiIconButton` `sizeSmall` sets `minWidth`/`width` **40**. Setting only
      `sx={{ width: 24, height: 24 }}` leaves `minWidth: 40` → a **40×24 rectangle** (looks like a wide grey
      chip). Always set `minWidth: 24` (and prefer `maxWidth: 24`) or `minWidth: 'unset'`, same as
      `ActionButton`. Do not rely on `size="small"` for a 24px Figma control.

    **Critical — `ice-Plus` on My Card “Add New” row** (`31293:462883` / nested `ice-Plus`):
    - Figma **outer** instance = **20×20**; **leaf** Group = **11×11** (inset 4.5); paint `#424242` →
      `color="iceGray.700"` (match the label). **Never `size={20}`** on the glyph — same outer≠leaf
      mistake as Close. Call site: `size={11}` + `customIcon={<Plus />}`. Do not use `text.primary`
      when Figma cites gray-700.

    **Critical — `ice-Plus` on Edit/Add card “Add another …” Text Buttons** (`18302:510070` Social,
    `18302:507392` Contact Details):
    - Figma **outer** instance = **14×14**; **leaf** `Group 1010106150` = **10×10** (inset 2); paint
      `#0073E6` → `color="primary.500"`. Gap to caption = **6px** (`spacing={1.5}` with theme
      `0.25rem` unit). **Never `size={14}`** on the glyph — that stretches the fill arms and reads as
      a heavier stroke. Call site: `size={10}` + `customIcon={<Plus />}`. Geometry is fill (not
      stroke); arm thickness is baked in the 10×10 path.

    For other icons: if Figma nests padding (40×40 control, 16×16 leaf) → control 40, `size={16}`.
    Do not treat “outer frame = always pass as size” when the leaf is smaller inside that frame.

    **Critical — Collab social brand SVGs** (`10384:304918` My Card footer; Edit Social `33284:438985`+):
    - Ship **circular Size=L** badges into `libraries/shared/assets/images/icons/social-network/ice-*.svg`
      (gradient IG, dark-circle white X, circular LI/YT/FB/WA/SC). Barrel `INSTAGRAM` / `X_ICON` / …
      must point at those `ice-*` files.
    - **False friends:** outline Instagram, bare black X glyph, rounded-square LinkedIn — do **not**
      restore these from a CleanShot of a broken build. Linked Figma nodes above are geometry authority.
    - Parent `input-outlined` MCP often returns remix **search-line** — always fetch the nested
      `ice-*` instance (`…;3331:31358`).

## The two icon folders — do not mix them up

| Folder                                  | Role                                                                              | Component shape                                                    |
| --------------------------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `libraries/shared/assets/icons/`        | **The DsIcon set.** UI glyphs tinted by `DsIcon color`, plus the two-tone set.    | `export const Name = () => <svg>…currentColor…</svg>` — zero props |
| `libraries/shared/assets/images/icons/` | Older illustration / multi-colour / brand set (spot colours, empty states, logos) | `export const Name = (props: SVGAttributes<SVGElement>) => …`      |

New work goes in `icons/`. Only put something in `images/icons/` when it is a fixed multi-colour
illustration that must never be tinted and is not a semantic tone pair.

The boundary moves in one direction. When a glyph gets a real DsIcon call site, it migrates
`images/icons/` -> `icons/` and the old barrel line is **removed**, not left alongside — that is what
happened to `Lock` and `Unlock`, which now exist in both folders but are exported only from `icons/`.
Re-exporting both would be a duplicate-name collision in the barrel.

## Naming convention

- **PascalCase, and the filename equals the exported const equals the import name.** This holds for
  all 125 icons today — keep it exact, it is what makes the barrel greppable.
- **No `Icon` suffix.** `Search`, not `SearchIcon`. The one exception is a name collision with the
  `images/icons/` set — `DownloadIcon` exists only because `Download` is already an illustration.
  Before naming, check both folders.
- **Noun first, modifier last:** `EmailRead`, `EmailUnread`, `EmailInbox`, `SoundOn`, `SoundOff`.
- **`Fill` suffix = the filled variant** of an outline sibling: `Flag` / `FlagFill`,
  `CircleCheck` / `CircleCheckFill`, `VoiceOn` / `VoiceOnFill`.
- **`Circle` prefix = the glyph inside a circle:** `CircleCheck`, `CircleCloseFill`,
  `CircleChevronRight`, `CircleWarningFill`.
- **`Ice` prefix = disambiguation from an MUI component of the same name.** `IceList` exists because
  `List` would collide with `@mui/material`'s `List`; likewise `IceEmojiPlus`. It also matches the
  Figma layer names, which are `ice-List` / `ice-Grid`.
  `Grid` is the **outlier that got this wrong** — it collides with MUI `Grid`, which `mui-layout-grid`
  mandates for every multi-column layout and which appears in ~90 files. `ListView.tsx` only escapes
  because it lays out with `Box`. Any file needing both must alias
  (`import { Grid as GridIcon } from '@ice-web-app/assets'`). Prefer `Ice…` for a new glyph whose
  natural name is an MUI export.
- **`Category` prefix = a picker tab glyph, not a content icon:** `CategoryPerson`, `CategoryLeaf`,
  `CategoryGear`, `CategorySparkles`, `CategoryPlanet`, `CategoryBeaker`, `CategoryChip`,
  `CategoryHierarchy`. These are the small tintable tabs above the Choose-a-Symbol grid; the large
  two-tone illustrations they switch between are named for the concept (`Solar`, `Robotics`).
- Derive the name from the `IceIconsEnum` key by PascalCasing it: `USER_CIRCLE_STICK` -> `UserCircleStick`,
  `CHEVRON_DOWN` -> `ChevronDown`. That keeps the migration mechanical and reviewable.
- Existing names are the shortlist — check `references/icon-catalog.md` before creating anything, so
  you reuse `Close` instead of adding `CloseSmall`.

## Component anatomy

Single-path glyph — the default shape:

```tsx
/** Figma ice-Grid (ToggleButtonGroup grid toggle glyph). */
export const Send = () => {
  return (
    <svg xmlns="http://www.w3.org/2000/svg" width="18" height="16" viewBox="0 0 18 16" fill="none">
      <path d="M…" fill="currentColor" />
    </svg>
  );
};
```

### Provenance JSDoc — required on every new icon

A one-line JSDoc naming the Figma origin. Path data is unreadable, so this comment is the only way a
reviewer can re-find the source node, and it is what makes the reuse gate's false friends
(`Draft` ~ `FrameSelect`, `AlignJustify` ~ `AlignLeft`) distinguishable at a glance:

```tsx
/** Figma State=Hidden card status glyph (Component 204 / Group 1010109015). */
/** Figma ice-List — bulleted list toggle glyph 14×11.68. */
/** Figma Choose-a-Symbol category tab — CategoryLeaf. Tintable currentColor. */
/** Figma Choose-a-Symbol recommended glyph — Analytics. Preserves multi-color fills. */
```

Name the frame or component (with the node id when you have it), and **state the paint mode when it
is not plain tintable** — `Preserves multi-color fills` and `Tintable currentColor` are the two
phrases in use. Do not restate the geometry; the paths already say that.

Multi-path glyph — repeat `<path>`, each with `fill="currentColor"` (see `Delete.tsx`, four paths).

One very long path — hoist it to an exported `<NAME>_ICON_PATH` const so the JSX stays readable
(`Bold.tsx`):

```tsx
export const BOLD_ICON_PATH = 'M0 15.42C0 15.7 …';

export const Bold = () => {
  return (
    <svg width="12" height="16" viewBox="0 0 12 16" fill="none" xmlns="http://www.w3.org/2000/svg">
      <path d={BOLD_ICON_PATH} fill="currentColor" />
    </svg>
  );
};
```

Many short paths of identical style — hoist to a plural `<NAME>_ICON_PATHS` const and map
(pattern used by the rich-text toolbar icons, e.g. `AlignCenter.tsx`):

```tsx
export const ALIGN_CENTER_ICON_PATHS = ['M…', 'M…'] as const;

export const AlignCenter = () => {
  return (
    <svg width="16" height="14.42" viewBox="0 0 15.9799 14.4202" fill="none" xmlns="http://www.w3.org/2000/svg">
      {ALIGN_CENTER_ICON_PATHS.map((d) => (
        <path key={d} d={d} fill="currentColor" />
      ))}
    </svg>
  );
};
```

Non-negotiables inside the file:

- **No props.** `DsIcon` owns size and colour; a props interface invites per-call-site drift.
- **Keep Figma's `viewBox` and intrinsic `width`/`height`**, fractional values included
  (`viewBox="0 0 15.9799 14.4202"`). Do not round to `24×24`.
- `fill="none"` on the `<svg>`, paint on the `<path>`.
- Stroked glyphs use `stroke="currentColor"` and keep Figma's stroke width. In JSX that prop is
  **camelCase** — `strokeWidth="0.3"`, not `stroke-width` (see `CategoryLeaf.tsx`, which fills and
  strokes with `currentColor` to thicken a hairline glyph).
- **Fill + stroke on the same path:** Figma exports often paint both `fill="#424242"` and
  `stroke="#424242"` (or white/white). On **clean** outline glyphs (`CameraOn`, `VoiceOn`), convert
  **both** to `currentColor` — dropping stroke leaves a vanishing hairline on light buttons. Paths
  that were fill-only stay fill-only; paths that were stroke-only stay stroke-only **with** Figma's
  `strokeWidth`.
- **Never ship `stroke="currentColor"` without `strokeWidth` from Figma.** Exploded MCP exports that
  set fill+stroke with no width default to SVG stroke `1` and look dashed/incomplete at 16–20px
  (ICE Calls hub status phones). Prefer fill-only after screenshot QA, or an explicit width — see
  `references/color-conventions.md` failure mode B.
- A multi-shape glyph exported from a Figma group keeps its `<g>` wrapper (see `Analytics.tsx`).
  Don't flatten it — it preserves the group's paint inheritance.

### Colour

**Full rules and the tone table: `references/color-conventions.md`. Read it before painting a glyph
anything other than `currentColor`.** The short version:

`DsIcon` sets `color: theme.palette[a][b] !important` on the wrapper, which `currentColor` inherits.
So `currentColor` is the mechanism that makes `color="error.500"` work at the call site, and a baked
hex silently kills theming, dark mode, and every state colour at once.

`pnpm icons:audit` classifies every icon into one of these four modes and reports the split. Two of
them are legitimate, two are bugs:

| Mode          | Count | Rule                                                                  |
| ------------- | ----- | --------------------------------------------------------------------- |
| **Tintable**  | 75    | `currentColor` only. **The default — assume this.**                   |
| **Two-tone**  | 35    | One tone's strong + `100` pair baked in, tone declared in a registry. |
| **Grey**      | 11    | Baked `iceGray` hex. Legacy bug — migrate when you touch the file.    |
| **Off-token** | 0     | A hex the palette does not own. Never acceptable; keep this at zero.  |

- **Neutral grey is never a reason to bake a hex.** `#757575` / `#424242` / `#E0E0E0` are just the
  default `iceGray` values — they mean "untinted", so they become `currentColor`. The 11 offenders
  (`Search`, the rich-text toolbar set, `VoiceOnFill`) are legacy; do not copy them.
- **Two-tone is only for the Choose-a-Symbol illustrations.** Both hexes must resolve to the same
  semantic tone (`primary` `#0062C3`/`#CCE3FA`, `info` `#0022FF`/`#CCD3FF`, `success`
  `#228E3B`/`#D4EDDA`, `warning` `#D96300`/`#FCE2CC`, `error` `#D33535`/`#FBD8D8`), and the icon must
  get a `tone` entry in `recommendedSymbols.tsx`. Adding one is a **two-file change**.
- **`color` on a two-tone icon is a silent no-op** — the baked hex wins. Don't pass it.
- Colour semantics belong at the call site, which is why `Lock` and `Unlock` are single tintable
  glyphs coloured `error.500` / `success.500` rather than two pre-coloured files.
- When a parent owns the state colour (`ToggleButton`, selected nav row), `DsIcon`'s `!important`
  blocks it. Hand control back with `sx={{ color: 'inherit !important' }}` and keep the comment
  saying why — and then do not also pass `color`.

### Size and aspect ratio

`DsIcon customIcon` renders a square box of `size` px with `& svg { width: 100%; height: 100% }`.
The SVG's own `preserveAspectRatio` keeps the artwork undistorted and centred inside that box.

- **Pass the exact Figma outer-frame px as a number:** `size={24}`, `size={20}`, `size={16}`. Read
  the linked instance metadata (`width` / `height` on `ice-Close`, not a guessed token). This is the
  established convention — numeric almost without exception. The `sm`/`md`/`lg`/`xl` tokens belong to
  the old font path (`xl` = **28** — a common false friend when migrating `size="lg"`/`"xl"`).
- **Never inflate past Figma** to compensate for how small the webfont looked. If the UI instance is
  24×24, ship `size={24}` (and a 24×24 hit target when Figma’s control is also 24). Bumping to 28 or
  wrapping in a leftover 32×32 `IconButton` is a migration bug — fix the mismatch, do not preserve it.
- Measure **outer box** and **leaf glyph** separately when Figma nests padding (e.g. 40×40 button with
  a 16×16 leaf → `IconButton` 40, `DsIcon size={16}`).
- For a **non-square** glyph that must fill its box edge to edge, use the explicit
  `width` / `height` props instead of `size`.
- Reach for `sx={{ '& svg': { width: N, height: N } }}` only when a parent constrains the box.

## Registering the icon

`libraries/shared/assets/index.ts` has a dedicated `./icons/*` block after the `./images/icons/*`
block. Add one line, **alphabetically inside that block**:

```ts
export * from './icons/UserCircleStick';
```

Nothing else to wire — `@ice-web-app/assets` maps to this barrel in `tsconfig.base.json`.

If the glyph is replacing one in `images/icons/`, **delete that export line in the same commit**.
Two `export *` lines exporting the same name is a barrel collision, and whichever wins is decided by
file order rather than intent.

A two-tone illustration needs a second registration: its `{ id, Icon, tone, size }` entry in
`recommendedSymbols.tsx`, where `size` is `ceil(max(w, h))` of the Figma glyph.

## Migrating a call site

```tsx
// ❌ BEFORE — icon font
<DsIcon icon="TRASH" size="md" color="error.main" />;

// ✅ AFTER — shared SVG
import { Delete } from '@ice-web-app/assets';

<DsIcon customIcon={<Delete />} size={20} color="error.main" />;
```

Conditional icons swap the component, not the string:

```tsx
// ❌ BEFORE
<DsIcon icon={isFlagged ? 'FLAG_FILL' : 'FLAG'} size={20} />

// ✅ AFTER
<DsIcon customIcon={isFlagged ? <FlagFill /> : <Flag />} size={20} />
```

Data-driven menus (`icon: 'CHECK'` in a config array) become `customIcon: <Check />` — change the
config's type from `keyof typeof IceIconsEnum` to `ReactNode` and update every producer of that
config in the same commit. `libraries/zones/collaborationZone/src/pages/email/components/sidebar/FolderItem.tsx`
is the reference implementation.

Do not delete keys from `libraries/shared/types/iceIconsEnum.ts` while any call site still reads them.

## Getting the glyph out of Figma

1. `get_metadata` on the frame to find the **glyph instance id** (`I…;…`).
2. `get_design_context` / `download_assets` on **that glyph child** — never the parent `IconButton`
   or card. Parents routinely export a stale nested remix asset that does not match the screenshot.
3. `curl -L` the returned `figma.com/api/mcp/asset/….svg` URL immediately into
   `temp/icons/<slug>.svg`. Those URLs expire in about 7 days — committing one ships a broken icon.
4. Keep the file: the reuse gate needs the raw export, and the export is the source of truth for the
   `viewBox` and intrinsic size.

If the user has not given a Figma link for the glyph yet, **stop and ask for it**. Approximating is
the single most expensive mistake in this workflow.

## The reuse gate — run this before creating anything

Pixel parity is the requirement, so "does this already exist" has to be a measurement. Name
similarity is not evidence: `TRASH` is the existing `Delete`, `PENCIL` is `Edit`, `CHECK_CIRCLE` is
`CircleCheck` — while `INFO_CIRCLE` is **not** `Info`, because `Info.tsx` is a bare `6×11` italic "i"
with no circle in it. Both mistakes are invisible from the name alone.

### Step 1 — geometry match (mandatory, cheap)

```bash
pnpm icons:match temp/icons/<slug>.svg
```

`scripts/icon-match.mjs` parses the export's path commands into absolute coordinates, normalises them
against the glyph bounding box, and compares that fingerprint to all icons in
`libraries/shared/assets/icons`. It reports:

| Verdict     | Meaning                                     | What to do                                                                |
| ----------- | ------------------------------------------- | ------------------------------------------------------------------------- |
| `IDENTICAL` | Same geometry, same `viewBox`               | **Reuse. Create nothing.** Swap the call sites to the existing component. |
| `RESCALED`  | Same geometry, different export size        | **Reuse.** Set `size` at the call site — do not add a second file.        |
| `NEAR`      | ≥97% coordinate agreement, aspect within 1% | Go to step 2. Never accept on the score alone.                            |
| `CANDIDATE` | Same shape counts, aspect within 5%         | Go to step 2.                                                             |
| no output   | Nothing above `CANDIDATE`                   | Genuinely new — create it.                                                |

Calibration, so you know what the scores are worth: across all 7,260 pairs in the existing 121-icon
set the tool reports **zero** false `IDENTICAL`, `RESCALED`, or `NEAR`. It does report 9 `CANDIDATE`
pairs — including `AlignJustify` ~ `AlignLeft` at **76.7%** agreement and **0.07%** aspect delta.
Those are two visibly different icons. That is exactly why `CANDIDATE` and `NEAR` are not a pass.

To compare two icons already in the set:

```bash
node scripts/icon-match.mjs --icon Delete --icon Edit
```

### Step 2 — pixel overlay diff (mandatory for `NEAR` / `CANDIDATE`)

Render the candidate and the Figma export at the same size and diff them. Do not eyeball this and do
not decide from a screenshot description.

1. Render the existing component in isolation at the Figma glyph's intrinsic px, screenshot it tight
   to the SVG box (Playwright MCP, per `playwright-verify-fix-loop`).
2. Rasterise the Figma export to the same pixel dimensions.
3. Diff:

```bash
python3 apps/e2e/agent-browser/scripts/figma-frame-pixel-diff.py \
  --figma temp/icons/<slug>-figma.png \
  --app   temp/icons/<slug>-candidate.png \
  --out   temp/icons/<slug>-diff.png \
  --threshold 32 --fail-ratio 0.01
```

Icons are small, flat, and have no text, so the tolerance is tighter than for a page frame: **≤1%
differing pixels is a reuse**, anything above is a new glyph. Report the ratio in your summary — a
reuse decision without a number is not a decision.

### Step 3 — decide, and say why

| Outcome                       | Action                                                                                           |
| ----------------------------- | ------------------------------------------------------------------------------------------------ |
| `IDENTICAL` / `RESCALED`      | Reuse. State the component name and the verdict.                                                 |
| Diff ≤1%                      | Reuse. State the diff ratio.                                                                     |
| Diff >1%, same family         | New file. State the ratio and what differs (stroke weight, fill vs outline, cap style, padding). |
| Rotation of an existing glyph | Reuse with `sx={{ transform: 'rotate(…) }}` — **only** if the diff on the rotated render passes. |
| No candidate                  | New file.                                                                                        |

Never report "looks like the same icon". Report the verdict, the score, or the diff ratio.

### Step 4 — only now create the file

Convert monochrome fills/strokes to `currentColor`, keep Figma's `viewBox` and intrinsic size, add
the provenance JSDoc, then follow the component anatomy above. If a Figma path has **both** fill and
stroke of the same monochrome token, emit **both** as `currentColor` — do not collapse to fill-only
(that ships a vanishing hairline on light backgrounds). If the glyph is a two-tone semantic
illustration, also add its `tone` entry to `recommendedSymbols.tsx` in the same commit.

`references/reuse-aliases.md` lists the known aliases, the rotation family, the reserved names in
`images/icons/`, and the documented false friends. Read it before step 1 to pick candidates fast —
it is a shortlist, not a substitute for the gate.

## Verify

```bash
pnpm nx run assets:type-check          # or the touched project
pnpm nx run <touched-project>:type-check
pnpm lint:staged
```

Then confirm at runtime, per `playwright-verify-fix-loop`:

- the glyph renders (an unregistered barrel export silently yields nothing),
- `color` still tints it — proof that `currentColor` survived,
- measured box matches the Figma px.

Guards before you claim a section is done:

```bash
# no font usages left in the section
rg -n 'DsIcon[^>]*\bicon=' libraries/zones/<zone>/src/pages/<section>

# no duplicate glyph landed in the shared set
pnpm icons:audit
```

`pnpm icons:audit` cross-compares every pair in the set and classifies how each icon paints. Four
things must hold:

| Check                             | Required state                                                    |
| --------------------------------- | ----------------------------------------------------------------- |
| `IDENTICAL` / `RESCALED` pairs    | **zero** — one appearing means the icon you added already existed |
| `off-token` hexes                 | **zero**                                                          |
| `grey` icons                      | 11 today, and must only ever go **down**                          |
| two-tone icons vs `tone` registry | every row `ok` — no `MISMATCH`, no `UNREGISTERED`                 |

## Migration status (mandatory)

Trackers are the queue agents pick from. Code without a status update is unfinished work.

| Tracker                             | Scope                                                                               |
| ----------------------------------- | ----------------------------------------------------------------------------------- |
| `docs/icon-refactor-inventory.md`   | Repo-wide keys + every call site (Part 3)                                           |
| `docs/collab-zone-icon-refactor.md` | Collab zone focus list + browser paths — keep in sync with inventory for zone files |

**Same-turn checklist** after each migrated call site (or batch of sites in one file):

1. Add `✅ DONE — …` under the file bullet; rewrite `KEY` → lines to the shipped `customIcon`.
2. Drop the file from **Unresolved / dynamic** (and any “manual” placeholder) if the font prop is gone.
3. If you created a shared icon, barrel-export it and refresh `references/icon-catalog.md` dims / count;
   run `pnpm skills:sync` when skill references change.
4. Never leave a DONE site still listing PascalCase guesses (`Trash`, `Website`) when the gate chose
   an alias (`Delete`, `IceLink`).

## Mistakes

| ❌ Mistake                                                            | Why it fails                                                                             |
| --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Creating an icon without running `pnpm icons:match`                   | Ships a near-duplicate; the two drift and parity depends which file a call site picked   |
| Migrating a call site but not marking `✅ DONE` in the trackers       | Next agent re-picks the same file; inventory/collab lists lie about remaining work       |
| Marking DONE with the Proposed PascalCase (`Trash`) not the alias     | Tracker teaches the wrong import — `TRASH` is `Delete`, social link is `IceLink`         |
| Deciding reuse from the name — `TRASH` -> a new `Trash`               | `Delete` already is that glyph; now there are two trash cans                             |
| Deciding reuse from the name — `INFO_CIRCLE` -> `Info`                | `Info.tsx` is a bare `6×11` "i" with no circle; the wrong glyph ships                    |
| Treating `NEAR` / `CANDIDATE` as a reuse                              | `AlignJustify` ~ `AlignLeft` scores 76.7% and they are different icons                   |
| "Looks like the same icon", with no number                            | Not a measurement; a parity claim needs the verdict or the diff ratio                    |
| A second file that differs only in size                               | `DsIcon size` is the size knob — `CLOSE_SMALL` is `Close` at a smaller `size`            |
| Adding a new `<DsIcon icon="X" />`                                    | Extends the font set we are removing; the glyph is not in the new design source          |
| Hand-drawing "close enough" circles and lines                         | Diff overlays expose stroke weight, fill-vs-stroke, and node count immediately           |
| Hardcoding `fill="#424242"`                                           | `DsIcon color` stops working; breaks dark mode and error/success states                  |
| Converting fill→`currentColor` but dropping Figma's stroke            | Hairline outline vanishes on white bg — half the glyph looks untinted (Call mic/camera)  |
| Shipping `stroke="currentColor"` with no `strokeWidth`                | SVG defaults stroke to 1 — exploded Figma paths look dashed/incomplete at 16–20px        |
| Replacing a linked Collab Zone call glyph with management-info phones | Different geometry (solid handset + corner arrow). User-linked nodes are authority       |
| “MCP export is shredded → use a cleaner phone from another file”      | Paint-convert the **same** paths; do not swap products. Document optical `strokeWidth`   |
| Omitting `color` on icons inside contained CTAs                       | `DsIcon` defaults dark grey on blue buttons (ArrowRight on “Go to Call” / “Rejoin”)      |
| Using `react-icons` / ad-hoc SVGs for product chrome                  | Shared `libraries/shared/assets/icons` only — no Font Awesome share etc.                 |
| One toast chrome / one check icon for recording started **and** saved | Started = outline `CircleCheck` + Banner; saved = `CircleCheckFill` + standard toast     |
| Copying a neutral grey from Figma instead of `currentColor`           | Grey means "untinted", not a colour decision — it is the default `iceGray` value         |
| Baking a two-tone pair without a `tone` entry in the registry         | Nothing pins the colour; the selected card border and the artwork drift apart            |
| Mixing tones in one two-tone glyph (`error.600` + `primary.100`)      | `icons:audit` reports `MISMATCH` — the pair must be one tone's strong + `100`            |
| Passing `color` to a two-tone icon                                    | Silent no-op; the baked hex wins and the next reader assumes it works                    |
| Shipping two pre-coloured glyphs (`LockRed`, `UnlockGreen`)           | One tintable glyph + `color` at the call site; otherwise `icons:audit` flags a duplicate |
| Omitting the provenance JSDoc                                         | Path data is unreadable — nobody can re-find the Figma node or tell near-twins apart     |
| `stroke-width="0.3"` in JSX                                           | React needs `strokeWidth`; the kebab-case prop is dropped and the hairline vanishes      |
| Naming a glyph `List` / `Grid` after an MUI export                    | Collides with `@mui/material` in any file needing both — use the `Ice` prefix            |
| Rounding `viewBox` to `0 0 24 24`                                     | Glyph no longer registers on the Figma grid; spacing around it drifts                    |
| Naming it `SearchIcon` / `IcSearch` / `search`                        | Barrel becomes ungreppable; the set is strictly `PascalCase` = filename = export         |
| Putting the icon in a feature `assets/` folder                        | Next zone that needs it duplicates it — violates shared-first reuse                      |
| Forgetting the `export * from './icons/…'` line                       | Import type-errors, or renders nothing if a stale duplicate name resolves                |
| Adding a props interface to the icon                                  | Size/colour drift per call site; `DsIcon` already owns both                              |
| Wrapping a non-square glyph in `size={24}` and expecting `24×h`       | `size` is a square box — use `width`/`height` for a non-square footprint                 |
| Inflating `size` / hit-target past Figma (24→28/`xl`, 24→32 button)   | Legacy webfont optical hack. SVG matches UI — keep outer frame px exact                  |
| Committing a `figma.com/api/mcp/asset/...` URL                        | Expires in ~7 days                                                                       |
| Swapping a config array's `icon` string without retyping it           | `keyof typeof IceIconsEnum` no longer holds; other producers of that config break        |
