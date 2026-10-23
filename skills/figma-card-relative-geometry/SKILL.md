---
name: figma-card-relative-geometry
description: Measure Figma vs app using card/frame-relative absolute X/Y tables. Catches header/content overlap and cumulative field drift that Diffchecker overlays mis-blame on chrome. Use with figma-implicit-spacing + playwright-figma-ui-diff.
---

# Figma Card-Relative Geometry

When Diffchecker still ghosts after “gaps look right”, build an **absolute position table inside the card/frame** — not viewport coords, not chrome.

Companion: **`figma-implicit-spacing`** · **`playwright-figma-ui-diff`** · **`playwright-verify-fix-loop`**.

## Why

1. Portal sidebars/top nav in a Diffchecker overlay create **fake global shift**.
2. Figma siblings can **overlap** (`header.h=72`, `content.y=68`). Local padding on Content is not the same as padding below the visible header.
3. Field height Δ (44 vs 48) **accumulates** down the form; abs Y tables catch it in one pass.

## Workflow

### 1. From `get_metadata`, compute card-absolute targets

For each field/row inside the modal/card:

```
absY = content.y + row.y
absX = content.x + row.x + cell.x
```

Example (scan business review card):

| Node        | absY | absX | w×h     |
| ----------- | ---- | ---- | ------- |
| First field | 88   | 20   | 536×48  |
| Row 2       | 156  | 20   | …       |
| Notes       | 428  | 20   | 1092×72 |
| Footer      | 630  | 0    | 1132×80 |
| Cancel      | 646  | 896  | 100×44  |

### 2. Playwright measure (card-relative)

```js
const card = document.querySelector('[data-test="…-card"]').getBoundingClientRect();
const rel = (el) => {
  const r = el.getBoundingClientRect();
  return {
    y: Math.round(r.top - card.top),
    x: Math.round(r.left - card.left),
    w: Math.round(r.width),
    h: Math.round(r.height),
  };
};
```

### 3. Compare table

| Check      | Figma abs | App abs | Δ   | Severity   |
| ---------- | --------- | ------- | --- | ---------- |
| field[0].y | 88        | ?       |     | Fail if ≥2 |

Fix **top-of-form offset** first (`formPaddingTop = firstFieldAbsY − headerHeight`), then field height, then gaps.

### 4. Only then pixel-diff

Crop to `[data-test="…-page"]` or card — never full portal. Script: `figma-frame-pixel-diff.py`.

## Mistakes

- Treating Content `p: 20` as CSS padding below a 72px header when Content.y overlaps
- Diffchecker with sidebars visible on one or both sides
- Fixing random `mt` on rows instead of the single top offset / field height root cause
