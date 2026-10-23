---
name: figma-structure-before-sx
description: Before styling third-party/MUI widgets to match Figma, compare Figma layer tree to library DOM order. If structure differs, replace slots/components — do not CSS-hack. Use when DateCalendar headers, Accordion, Tabs, or any library chrome fails visual parity.
---

# Figma Structure Before `sx`

Companion: `figma-design-system-parity` · `figma-pixel-parity-check` · `ice-calls-calendar-layout` · `mui-sx-theme-callback`.

## Failure class (real incident)

`DsMiniCalendar` header vs Figma Row `25602:477718`:

| Source                 | Order                                                                                        |
| ---------------------- | -------------------------------------------------------------------------------------------- |
| **Figma**              | `[prev 44][month flex][next 44]` inside centered **218px** cluster                           |
| **MUI default**        | `[month label][ArrowSwitcher: prev + next]`                                                  |
| **Broken CSS attempt** | styled ArrowSwitcher `width: 218; space-between` while label stayed outside → `Month < …… >` |

Screenshot of our build showed month + left chevron grouped left, right chevron far right. Figma/UI reference showed month centered between arrows.

## Failure class 2 — clipped / scrolling day grid (`25602:477733`)

| Source          | Spec                                                                                       |
| --------------- | ------------------------------------------------------------------------------------------ |
| **Figma Dates** | height **200** = pt **12** + 5×**36** + pb **8**                                           |
| **Broken A**    | `minHeight: 180` on slide (= 5×36 **without** padding) + tight overflow → half-circle clip |
| **Broken B**    | `minHeight: 236` (pad + 6×36) to avoid clip → overall height **≠ 302**                     |

## Failure class 4 — wrong overall frame height

Figma ReactDatePicker is **`345×302`**. Always recompute:

`52 + 24 + 20 + 6 + 200 = 302`

If any child is “fixed for MUI convenience” (6-week slide, `VIEW_HEIGHT` 336, `height: auto`), re-check the **root** height against Figma before shipping.

## Failure class 3 — missing Figma flex `gap` (vertical rhythm)

Figma ReactDatePicker `25602:477717`:

```
root: gap 24  →  [header 52] [body]
body: gap 6   →  [weekdays 20] [Dates pt12…pb8]
```

**Broken:** only styled Dates `pt`/`pb`, never set root/body `gap` → header↔weekday spacing drifts (too tight or inflated by leftover MUI margins / PickerViewRoot height).

**Fix:** explicit `gap` on `.MuiDateCalendar-root` (24) and `.MuiDayCalendar-root` (6); lock header `52` and weekday row `20`; `maxHeight: 'none'`.

## Mandatory checks before styling library chrome

1. Open the **exact Figma node** (not only the parent frame screenshot).
2. Read **child layer order + widths + heights + padding + parent `gap`** (cluster width, button size, Dates frame height formula, flex gaps between sections).
3. Inspect the **library default DOM / slot tree** (MUI source or DevTools).
4. Decide:

```
Figma tree ≅ library tree?
├─ YES → token/sx/spacing parity on existing nodes
└─ NO  → custom slot / custom subcomponent that builds Figma tree
         (do NOT invent CSS order with flex tricks on the wrong parents)
```

5. Diff **your running UI screenshot** against Figma before calling the step done.
6. For grids/calendars: verify **last row + selection chrome** are fully visible (no clip, no inner scrollbar).
7. For stacked chrome: write down **gap vs padding** separately — padding inside a child is not the gap above it.

## Do

- Prefer `slots.calendarHeader` / other MUI slots when structure must match Figma
- Map Figma sizes exactly (here: cluster `218`, nav `44`, header `52`, root `gap 24`, body `gap 6`, typography `subtitle1`)
- Size slide/viewport from Figma math: **padding + rows×rowHeight + padding** (never omit padding)
- After fixing a subsection, **re-sum to the Figma root size** (mini calendar = **302**)
- Document the Figma node id next to the custom slot

## Do not

- Assume “arrows + label” means the library already nests them the same way
- Hide spacers / `space-between` an ArrowSwitcher and hope the label “joins” the cluster
- Treat a parent-frame screenshot as enough to implement a nested header/control chrome
- Ship without comparing your build screenshot to the Figma node screenshot
- Set calendar `minHeight` to “rows × day size” while also adding `pt`/`pb` inside that box
- Inflate Dates to 6 weeks when Figma root height is **302** (Dates is **5×36 + pads = 200**)
- Use `overflow: hidden` on mini-calendar surfaces that must show full circular day selection **without** matching Figma Dates height first
- Treat Dates `pt` as the only vertical spacing control — always check parent **`gap`** on the Figma frame

## Reference fix

`libraries/shared/designSystem/src/components/calendar/DsMiniCalendar.tsx` — `MiniCalendarHeader` via `slots.calendarHeader`; overall **302**; Dates **200** / 5 weeks.
