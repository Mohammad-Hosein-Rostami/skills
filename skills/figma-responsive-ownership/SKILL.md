---
name: figma-responsive-ownership
description: Measured responsive matrix for desktop-only Figma — columnAlign for tables, mandatory evaluate at every viewport, forbidden weak Pass claims like “no H-scroll”. Never stop at 1530×920 screenshots.
---

# Figma Responsive Ownership (desktop-only designs)

Companion: **`figma-page-shell-layout`** · **`mui-layout-grid`** · **`figma-absolute-layout-safety`** · **`playwright-agent-eyes`** · **`playwright-verify-fix-loop`** · **`figma-feature-delivery`**.  
Invoked as stage 7 of `fullstack-feature-delivery-loop`.

## Core truth

| Fact                                | Implication                                                             |
| ----------------------------------- | ----------------------------------------------------------------------- |
| 1530×920 is agent parity size       | Necessary; **never sufficient**                                         |
| “No page H-scroll” is a weak signal | **Never** enough to claim Pass                                          |
| Screenshots without measure         | **Incomplete** — must `getBoundingClientRect`                           |
| One screenshot per width            | Insufficient — require viewport + full-page + scroll/container coverage |
| Only <=1530 widths checked          | Insufficient — must include **1920 and 2400** desktop checks            |
| Tables/heatmaps                     | Must report **columnAlign maxΔ** per viewport or **Fail**               |

**Do not claim responsive Pass** unless the **Required verdict block** below is filled with real numbers from Playwright evaluate.

## Forbidden Pass claims (instant Fail)

Any of these **alone** (or as the only evidence) = **Fail** the responsive stage:

- “Responsive 375→1920 (no page H-scroll) Pass”
- “No overflow / looks fine”
- “Screenshots look OK at phone/tablet”
- “Matrix done” without per-viewport measurements
- Measuring **only** 1530 carefully and glancing other widths
- Measuring **only** header column widths (must include body + footer rows)
- Capturing only the scrolled middle of page (missing top chrome/context)
- Skipping >1530 widths (1920/2400) and still claiming responsive Pass

## Screenshot coverage protocol (MANDATORY)

At each required viewport, capture all of these:

1. **Viewport-top shot** at scrollY=0 (captures header/tabs/filter chrome).
2. **Full-page shot** (`fullPage: true`) to include the complete vertical page.
3. **Viewport-bottom context shot** if page is long:
   - scroll to ~50% and ~100%, capture viewport shots.
4. **Container scroll coverage** for internal scroll regions:
   - If container has `scrollHeight > clientHeight`: capture top and bottom of that container.
   - If container has `scrollWidth > clientWidth`: capture left and right edges.

Minimum evidence per viewport:

- top viewport PNG
- full-page PNG
- any required middle/bottom/container PNGs

Per-viewport visual diff checklist (must be written, not implied):

1. Top chrome (back/breadcrumb/tabs/filters) is present and unclipped.
2. Main content rows/columns remain aligned and readable.
3. Bottom/last-section content exists (not skipped by mid-scroll capture).
4. Fixed/sticky overlays (nav, toolbars) do not hide critical content.

If any required capture is missing, that viewport is **Fail**.

## When the UI is a table / heatmap / multi-row columns

Treat as table if any of: heatmap, sticky rails, per-row flex columns, `data-test` like `*-report-table`, shared header+body+footer columns.

### Known failure (must catch)

MI Report CleanShot: at non-1530, **jagged vertical rails** / heat cells offset vs header while 1530 looks perfect. Often from per-row flex + sticky Pupils/Total. Agents that only check page H-scroll **miss this**.

### Mandatory measure (every matrix viewport)

1. `browser_resize` to the viewport.
2. Set table `scrollLeft = 0`.
3. Collect cells from **header, ≥1 parent row, ≥1 different parent (or flat row), ≥1 child if any, footer**.
4. For each column index, compute `max(left) - min(left)` and `max(width) - min(width)` across those rows.
5. **Fail viewport** if `maxLeftΔ > 1` OR `maxWidthΔ > 2`.
6. Save PNG; **Read** the PNG for 1280 and 899 (and any failed width) and explicitly note whether rails look continuous.

Evaluate template:

```js
() => {
  const table = document.querySelector('[data-test="mi-report-table"]'); // adapt
  if (!table) return { error: 'no-table' };
  table.scrollLeft = 0;
  const rowEls = [/* header, parentA, parentB, child, footer */].filter(Boolean);
  const colCount = 9; // adapt
  const lefts = Array.from({ length: colCount }, () => []);
  const widths = Array.from({ length: colCount }, () => []);
  for (const row of rowEls) {
    const cells = [...row.children];
    for (let i = 0; i < colCount; i++) {
      const r = cells[i].getBoundingClientRect();
      lefts[i].push(Math.round(r.left));
      widths[i].push(Math.round(r.width));
    }
  }
  const maxLeftΔ = Math.max(...lefts.map((xs) => Math.max(...xs) - Math.min(...xs)));
  const maxWidthΔ = Math.max(...widths.map((xs) => Math.max(...xs) - Math.min(...xs)));
  return { vw: innerWidth, maxLeftΔ, maxWidthΔ, pass: maxLeftΔ <= 1 && maxWidthΔ <= 2 };
};
```

### Implementation guidance (after Fail — not instead of measure)

- Prefer shared `grid-template-columns` tracks over independent per-row flex.
- Avoid sticky left/right rails unless Figma shows them **and** matrix still passes `columnAlign`.
- Narrow pane → scroll **whole** table; do not crush columns below Figma mins.

## Required viewport matrix (all)

| Pass | Viewport  | Must measure                                         |
| ---- | --------- | ---------------------------------------------------- |
| A    | 1530×920  | parity + columnAlign                                 |
| B    | 375×667   | overflow + columnAlign + stacked chrome              |
| C    | 430×932   | columnAlign                                          |
| D    | 768×1024  | columnAlign                                          |
| E    | 899×800   | columnAlign (common sticky/flex break)               |
| F    | 1280×800  | columnAlign (common Report break)                    |
| G    | 1920×1200 | proportional cols + columnAlign                      |
| H    | 2400×1350 | xxxl/wide desktop wrapping + edge pins + columnAlign |

Also: one slow resize sweep ~320→2400; stop on first columnAlign fail. Re-verify 1530 after fixes.

## Required verdict block (copy into final deliverable)

Agents **must** paste this filled (or Fail):

```text
## Responsive ownership — [route]
Table/heatmap UI: yes|no

| Viewport | pageHOverflow | columnAlign maxLeftΔ | maxWidthΔ | topPNG | fullPagePNG | extraScrollPNG(s) | Pass? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1530×920 |  |  |  |  |  |  |  |
| 375×667 |  |  |  |  |  |  |  |
| 430×932 |  |  |  |  |  |  |  |
| 768×1024 |  |  |  |  |  |  |  |
| 899×800 |  |  |  |  |  |  |  |
| 1280×800 |  |  |  |  |  |  |  |
| 1920×1200 |  |  |  |  |  |  |  |
| 2400×1350 |  |  |  |  |  |  |  |

Overall responsive: Pass | Fail
Fail reasons: …
```

Rules:

- If `Table/heatmap UI: yes` and any `columnAlign` cell is blank → **Fail** (incomplete).
- If any `maxLeftΔ > 1` → **Fail** (jagged rails / heat offset).
- If top/fullPage/required scroll coverage screenshots are missing → **Fail**.
- If 1920 or 2400 row is missing for desktop-responsive UI → **Fail**.
- If Overall is Pass but table is yes and only pageHOverflow was checked → **invalid**; treat as Fail.

## Non-table UIs

Still require per-viewport: page overflow, fractional grid cards, abs overlap, stacked toolbars, edge CTA at 1920. Screenshots + boxes; no weak one-liner Pass.

## Related

- `fullstack-feature-delivery-loop` — stage 7 and the required deliverable format
- Failure class in `figma-feature-delivery`: Report jagged rails off-1530
- Include the filled verdict block in the final task summary
