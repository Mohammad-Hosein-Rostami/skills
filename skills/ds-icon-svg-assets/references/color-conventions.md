# Icon colour conventions

How a glyph in `libraries/shared/assets/icons/` is allowed to paint itself, and how that interacts
with `DsIcon color`. Measured against the current 121-icon set with `pnpm icons:audit`.

## The three paint modes

| Mode          | Count | Paints with                           | Responds to `DsIcon color`? | Verdict                                        |
| ------------- | ----- | ------------------------------------- | --------------------------- | ---------------------------------------------- |
| **Tintable**  | 75    | `currentColor` only                   | Yes                         | **Default. Use this unless proven otherwise.** |
| **Two-tone**  | 35    | One semantic token pair, baked as hex | **No** — hex wins           | Legitimate only for the illustration set       |
| **Grey**      | 11    | `iceGray` hex baked in                | **No** — hex wins           | Bug. Migrate to `currentColor`.                |
| **Off-token** | 0     | A hex the palette does not own        | No                          | Bug. Never acceptable.                         |

`pnpm icons:audit` prints these four buckets and will name the offenders. Grey must trend to zero;
off-token must stay at zero.

## Why `currentColor` is the default

`DsIcon` resolves the `color` prop against the theme and sets it on the wrapper as `!important`:

```tsx
sx={{ color: `${theme.palette[color.split('.')[0]][color.split('.')[1]]} !important` }}
```

`currentColor` inside the SVG inherits that. It is the entire mechanism by which
`color="error.500"` tints a glyph, and it is why a baked hex silently kills theming, dark mode, and
every state colour at once. `colorSchemesType` is a union of **dotted** paths only
(`'primary.500'`, `'iceGray.700'`), so always pass two segments.

## Two-tone semantic illustrations

The Choose-a-Symbol set (`RecommendedSymbolPicker`) is deliberately **not** tintable: each glyph is a
small illustration whose colour carries the category's tone. Those glyphs bake exactly two hexes —
a strong shade for the artwork and a `100` tint for the highlights — and both must resolve to the
same semantic tone.

| Tone      | Strong (artwork) | Token         | Tint (highlight) | Token         |
| --------- | ---------------- | ------------- | ---------------- | ------------- |
| `primary` | `#0062C3`        | `primary.700` | `#CCE3FA`        | `primary.100` |
| `info`    | `#0022FF`        | `info.500`    | `#CCD3FF`        | `info.100`    |
| `success` | `#228E3B`        | `success.700` | `#D4EDDA`        | `success.100` |
| `warning` | `#D96300`        | `warning.600` | `#FCE2CC`        | `warning.100` |
| `error`   | `#D33535`        | `error.600`   | `#FBD8D8`        | `error.100`   |

Neutral hexes are **not** on this list on purpose — a grey glyph has no tone, so it must be
`currentColor`:

| Hex       | Token         | Where it wrongly appears today                            |
| --------- | ------------- | --------------------------------------------------------- |
| `#757575` | `iceGray.500` | the 9 rich-text toolbar icons (`Bold`, `AlignLeft`, …)    |
| `#424242` | `iceGray.700` | `Search`                                                  |
| `#E0E0E0` | `iceGray.200` | `VoiceOnFill` (mixed with `currentColor` — worst of both) |

### The tone must be declared, not just baked

A baked hex is invisible to the call site, so the pairing is pinned by a registry.
`libraries/zones/collaborationZone/src/pages/collabGroups/components/recommendedSymbols.tsx`
declares a `tone` per symbol, and the picker derives the card chrome from it:

```tsx
export type SymbolTone = 'primary' | 'info' | 'success' | 'warning' | 'error';

{ id: 'Solar', Icon: Solar, tone: 'error', size: 63 }
```

```tsx
border: selected ? `6px solid ${theme.palette[symbol.tone][500]}` : '6px solid transparent',
bgcolor: (theme) => theme.palette[symbol.tone]['opacity-16'],
```

If the baked pair and the declared `tone` disagree, the selected border and the artwork inside it
render in different colours. `pnpm icons:audit` cross-checks every two-tone icon against this
registry and reports `ok` / `MISMATCH` / `UNREGISTERED` — all 35 are `ok` today, keep it that way.

Two consequences worth internalising:

- **Adding a two-tone icon means editing two files**: the glyph, and the registry entry that
  declares its tone and preview `size`.
- **Two-tone glyphs render bare**, not through `DsIcon`: `<SymbolIcon />` inside a `Box` that scopes
  `& svg { max-width, max-height }`. Passing `color` to a two-tone icon is a silent no-op, so don't.

## Deciding a glyph's mode from Figma

Ask what the fill means, not what it looks like:

1. Does the Figma glyph change colour across states, or is it drawn once per semantic colour
   (`error` lock vs `success` unlock)? → **tintable**, one component, `color` at the call site.
2. Is it a flat neutral grey (`#757575`, `#424242`, `#E0E0E0`)? → **tintable**. The grey is the
   default `text.primary`/`iceGray` value, not a decision.
3. Is it a multi-shape illustration whose two colours are a strong + tint pair of one semantic
   tone? → **two-tone**, and add the registry entry.
4. Anything else multi-colour and fixed (brand marks, empty states) → it does not belong in
   `icons/` at all. It goes in `images/assets/images/icons/` with a props signature.

`figma-interactive-state` covers reading the intended state colour off Figma's fills rather than
guessing it from the screenshot.

## Fill + stroke must both become `currentColor`

Figma vector exports frequently set **both** `fill` and `stroke` to the same grey (or white) on one
path. That stroke is not decorative — it is half the visible weight of the glyph. When converting:

| Figma path paint                         | Convert to                                             |
| ---------------------------------------- | ------------------------------------------------------ |
| `fill="#424242"` only                    | `fill="currentColor"`                                  |
| `stroke="#424242"` only                  | `stroke="currentColor"` (+ keep `strokeWidth`)         |
| `fill` **and** `stroke` same monochrome  | **both** `currentColor` — never drop stroke            |
| `fill="white"` + `stroke="#424242"` etc. | both → `currentColor` for tintable call-control glyphs |

**Failure mode A (ICE Calls mic/camera):** converter kept only `fill="currentColor"` and dropped an
intentional stroke. On a white Lit button (`iceGray.50` bg + `iceGray.700` icon), the mic capsule
rendered as a faint hairline while the cradle (fill-only path) went solid black; the camera was
nearly invisible. Keep both paints as `currentColor` for clean outline glyphs (`CameraOn`,
`VoiceOn`).

**Failure mode B (incomplete / dashed at 16–20px):** Figma MCP export has `fill`+`stroke` on
**exploded / boolean-outline** paths **without** an explicit `stroke-width`. SVG then defaults
`strokeWidth` to `1`, which shreds the glyph into dashed fragments at hub sizes. This is **not** the
same as failure mode A — do not blindly keep a bare `stroke="currentColor"`.

| Export situation                                                        | Convert to                                                                                             |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Clean outline path, Figma sets fill+stroke (CameraOn / VoiceOn)         | both `currentColor`; preserve Figma `strokeWidth` when present                                         |
| Stroke-only tick/arrow with Figma `stroke-width`                        | `stroke="currentColor"` + that `strokeWidth` (camelCase in JSX)                                        |
| Fill+stroke on messy/exploded paths, **no** Figma stroke width          | Prefer **fill-only on those same paths** after screenshot QA at call-site size; never ship bare stroke |
| Stroke-only overlay (arrow / X) with no Figma stroke width              | `stroke="currentColor"` + documented optical `strokeWidth` (e.g. `1.2`) — keep the path `d`            |
| Fill-only already matches Figma screenshot at `size={16}` / `size={20}` | Keep fill-only; document why in the file JSDoc                                                         |

**Paint conversion ≠ glyph substitution.** Failure mode B authorises changing _how_ the linked
paths paint (`currentColor`, drop bare stroke, add optical width). It does **not** authorise
replacing Collab Zone hub call status paths with management-info (or any other file’s) solid phone
SVGs — that was a real regression (`CallIncoming` / `CallOutgoing` / `CallNoAnswered`).

Never ship `stroke="currentColor"` without an explicit `strokeWidth` taken from Figma (or a
documented optical substitute after visual QA). Always visual-check at the call-site size after any
fill+stroke conversion.

`DsIcon color` only works if every painted attribute inherits `currentColor`.

**`DsIcon` + nested custom SVG:** MuiSvgIcon defaults `fill: currentColor`, which **inherits** into
the nested icon and overrides presentation `fill="none"` on stroke-only marks (arrows / X). The
customIcon branch sets `fill: 'unset'` and `overflow: 'visible'` so path-level attributes win and
edge strokes are not clipped.

## Colour at the call site

Pass a token, never a literal. Real examples from `collabGroups`:

```tsx
<DsIcon customIcon={<Lock />}   color="error.500"   size={23} />
<DsIcon customIcon={<Unlock />} color="success.500" size={19} />
<DsIcon customIcon={<Ghost />}  color="iceGray.500" size={20} />
```

The semantics live at the call site, which is why `Lock` and `Unlock` are single tintable glyphs and
not `LockRed` / `UnlockGreen`.

### CTA / contained buttons

`DsIcon` defaults to a dark grey when `color` is omitted. On a primary/contained `DsButton` that
reads as white text, an untinted icon stays dark on blue. Always set colour (or inherit) for icons
inside buttons:

```tsx
// Contained / primary filled — match white label
<DsIcon customIcon={<ArrowRight />} size={14} color="iceGray.50" />

// Outlined — match primary label
<DsIcon customIcon={<ArrowRight />} size={14} color="primary.500" />

// Or hand colour back to the button
<DsIcon customIcon={<ArrowRight />} size={14} sx={{ color: 'inherit !important' }} />
```

Same pattern as `HangUp` on error hang-up buttons (`color="iceGray.50"`).

### Letting a parent own the colour

`DsIcon` writes `color: … !important`, so a parent's `:hover` / `.Mui-selected` colour cannot win by
specificity. When the parent legitimately owns the state colour — a `ToggleButton`, a selected nav
row — hand control back explicitly:

```tsx
// Inherit selected white / unselected gray from ToggleButton (DsIcon default color is !important)
const toggleIconSx = { color: 'inherit !important' } as const;

<DsIcon customIcon={<Grid />} size={14} sx={toggleIconSx} />;
```

This is the sanctioned escape hatch (`ListView.tsx`). Two rules: keep the comment explaining why
`!important` is there, and do not also pass `color` — pick one owner.

### Driving tone from data

When the colour comes from a status or a category, resolve the token in a map and pass it through.
Never branch on a hex:

```tsx
// ✅ token in, token out
<DsIcon customIcon={<CircleCheckFill />} color={STATUS_TONE[status]} size={20} />

// ❌ hex bypasses the theme
<DsIcon customIcon={<CircleCheckFill />} sx={{ color: '#228E3B !important' }} />
```

## Mistakes

| ❌ Mistake                                                   | Why it fails                                                                              |
| ------------------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| Dropping Figma `stroke` when converting to `currentColor`    | Hairline vanishes on light bg — glyph looks half-tinted or invisible                      |
| Copying Figma's `fill="#757575"` into a new glyph            | `DsIcon color` becomes a no-op; dark mode and error/success states break                  |
| Two glyphs, `IconRed` and `IconGreen`, same geometry         | One tintable glyph + `color` at the call site — otherwise `icons:audit` flags a duplicate |
| Adding a two-tone icon without a registry entry              | Nothing pins its tone; card border and artwork drift apart                                |
| Two-tone glyph mixing tones (`error.600` + `primary.100`)    | `icons:audit` reports `MISMATCH`; the pair must be one tone's strong + `100`              |
| Passing `color` to a two-tone icon                           | Silent no-op — the baked hex wins; readers assume it works                                |
| Leaving `currentColor` alongside a baked hex (`VoiceOnFill`) | Half the glyph tints and half does not, which reads as a rendering bug                    |
| `sx={{ color: '#… !important' }}` at the call site           | Same theming loss as baking it in, just harder to grep                                    |
| Using `color="primary"` (one segment)                        | `colorSchemesType` is dotted-only; resolves to `undefined`                                |
| `color: 'inherit !important'` with no comment                | Next reader deletes the `!important` and the parent state colour silently stops           |
| A hex outside the palette table above                        | Off-token by definition; it will not follow a theme change                                |
