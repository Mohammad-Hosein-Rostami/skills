---
name: missing-ds-icon-anchors
description: NARROW FALLBACK — Comment Anchors ANCHOR markers for a Figma glyph you cannot obtain yet. For all normal icon work load ds-icon-svg-assets instead, which puts the SVG in libraries/shared/assets/icons.
---

# Missing DS Icon Anchors

> **Mostly superseded by `ds-icon-svg-assets`.** A glyph with no component is no longer a blocker
> needing a placeholder: create it in `libraries/shared/assets/icons` and render it via
> `DsIcon customIcon`. Waiting on the icon font is not part of the flow anymore.
>
> This skill still applies in exactly one case: you need the UI to ship **before** the Figma node is
> available, so the SVG cannot be authored correctly yet. Then leave a feature-local placeholder plus
> an `ANCHOR:` marker, and say so in the summary.

Use with **`ds-icon-svg-assets`**, `figma-design-system-parity` / `figma-pixel-parity-check`,
and `skill-first-governance`.

## When

- The Figma node for the glyph is not obtainable yet and the UI must ship
- User asks to leave Comment Anchors markers for missing icons

## Workflow

1. Confirm no `IceIconsEnum` match (search `libraries/shared/types/iceIconsEnum.ts` and real `DsIcon` usage).
2. **Download the exact Figma SVG paths** — do **not** freehand-approximate the glyph:
   - Call `get_design_context` on the **icon glyph child** (e.g. `ice-collapse-arrow left`), **not** the parent `IconButton` / card. Parent instances often export a **stale/wrong** nested remix asset (e.g. `filter-2-line`) that does not match the visible screenshot.
   - Prefer `get_metadata` first to find the glyph instance id (`I…;…`), then `get_design_context` / `download_assets` on that id.
   - `curl -L` the returned `figma.com/api/mcp/asset/….svg` URLs promptly (they expire ~7 days) and **inline the path data** into a local React SVG component — never commit the live URL as `img src`.
   - Convert fills/strokes to `currentColor` and pass the token at the call site. A grey Figma fill means "untinted" — it is **not** an exception. The only glyphs allowed to bake hex are two-tone semantic illustrations (one tone's strong + `100` pair, tone declared in a registry); see `ds-icon-svg-assets/references/color-conventions.md`. A state difference like sidebar-closed vs opened is one glyph plus two `DsIcon color` tokens, not two baked hexes.
   - Preserve Figma `viewBox` / intrinsic width×height (e.g. `20×16.33`, not a guessed `24×24` redraw).
3. Place under feature `assets/` (e.g. `BusinessCardFilledIcon.tsx`) — **temporary only**. The
   permanent home is `libraries/shared/assets/icons` (`ds-icon-svg-assets`); promote it there as soon
   as the Figma node lands.
4. Add the required Comment Anchors line **above** import or usage:

```ts
// ANCHOR: missing-ds-icon/<ICON_ID> — <short Figma description> (node <nodeId>). Temporary local SVG until IceIconsEnum + DsIcon exist.
```

5. Render with `DsIcon customIcon={<LocalIcon />} size={N}` for square glyphs. (`DsIcon` has no
   `type` prop — that was an older API.)
   - **Critical:** `size={N}` forces a **square** `N×N` box. Non-square Figma glyphs like
     business-card **23.88×18.77** need the explicit `width` / `height` props instead, or render
     `<LocalIcon />` inside a coloured well so its native `viewBox` wins.
6. Inform the user: icon ID, Figma node, temporary path, that Comment Anchors can list `missing-ds-icon/`.

## Common mistakes (repeat failures)

| ❌ Mistake                                                          | Why it fails                                                                   |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| Hand-drawing circles/lines that “look like” Figma                   | Diff overlays show wrong stroke weight, filled vs stroked geometry, node count |
| Replacing a **filled** network share path with 3 circles + 2 bars   | Wrong glyph silhouette vs Ice/share-filled                                     |
| Scaling a complex viewfinder into a single guessed `rect` + L-paths | Corner radius / stroke 1.724 vs 1.75 drifts under slider diffs                 |
| Wrapping non-square Figma SVG in `DsIcon size={24}`                 | Forces 24×24 — squash/stretch; measure shows `h≈24` instead of Figma `18.77`   |
| Composing multi-stroke icons with approximate `%` transforms        | Prefer Figma inset → **px in a fixed box** (e.g. 100×100) from metadata        |
| Leaving hard-coded `#28A745` / `#EA3B3B` fills                      | Breaks DS tinting via parent `color`                                           |
| Committing live `figma.com/api/mcp/asset/...` URLs                  | Expire in ~7 days; icons break / show wrong glyphs                             |
| `get_design_context` only on parent `IconButton`                    | Nested slot may export wrong remix SVG; dig to glyph instance + verify vs shot |

## Finding later

In VS Code with **Comment Anchors** (`exodiusstudios.comment-anchors`): open the Anchors sidebar / workspace search for `missing-ds-icon/`.

Optional custom tag (settings):

```json
{
  "tag": "MISSING_DS_ICON",
  "iconColor": "orange",
  "highlightColor": "#FFA500",
  "styleComment": true
}
```

Still keep the `ANCHOR: missing-ds-icon/...` line as the team standard.

## Do not

- Commit ephemeral `figma.com/api/mcp/asset/...` URLs
- Skip the ANCHOR comment
- Add icons to `IceIconsEnum` without the real font glyph / team icon process
