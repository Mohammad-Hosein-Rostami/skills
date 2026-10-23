---
name: figma-implicit-spacing
description: Derive Figma spacing when MCP omits gap/padding, returns computed auto values, or slots miss flex props. Use with get_metadata coordinate math + Playwright box measurement to verify runtime.
---

# Figma Implicit Spacing

Companion skills: **`figma-pixel-parity-check`** · **`figma-design-system-parity`** · **`figma-page-shell-layout`** · **`playwright-verify-fix-loop`**.  
Rules: `figma-pixel-parity.mdc` · `design-system-and-styling.mdc`.

## When

- `get_design_context` lists `gap` / padding but Dev Mode shows none (computed “auto” value)
- `get_metadata` has `x`/`y`/`width`/`height` but no auto-layout props
- Parent uses **Space between** / **Gap: Auto** — API may return stale `itemSpacing`
- Slot / instance children missing flex props in MCP output
- Agent guessed `Stack spacing={6}` or MUI default 8px instead of Figma px
- Playwright `browser_snapshot` with `boxes: true` disagrees with implemented spacing

## Why MCP lies (do not trust gap blindly)

| Source issue                 | What happens                               | Agent mistake              |
| ---------------------------- | ------------------------------------------ | -------------------------- |
| Unset gap in Figma           | REST/MCP returns **computed** px (e.g. 18) | Hardcode fake gap          |
| `SPACE_BETWEEN` / gap Auto   | Stale `itemSpacing` in API                 | Use as CSS `gap`           |
| Slots                        | Flex props omitted on slot wrapper         | Re-invent layout           |
| Absolute-positioned children | No auto-layout on parent                   | Miss sibling gaps entirely |
| Nested frames                | Each level has own padding + gap           | Only read outer frame      |

Official Figma implement skill: use **`get_metadata`** for structure, then targeted **`get_design_context`** on nodes that own spacing — not one giant paste.

## Mandatory workflow

### 1. Map structure first (`get_metadata`)

For the frame under implementation, list **every child** with:

```
id, name, type, x, y, width, height  (relative to parent)
```

Build a small tree. Identify which nodes are **siblings under the same layout parent** (the node whose spacing you are implementing).

### 2. Derive spacing from coordinates (truth when gap is missing)

All math uses **parent-relative** metadata coords (same units as Figma px).

**Column stack (vertical siblings):**

```
gap_i = child[i+1].y − (child[i].y + child[i].height)
```

**Row (horizontal siblings):**

```
gap_i = child[i+1].x − (child[i].x + child[i].width)
```

**Padding (single child or first/last in stack):**

```
paddingTop    = firstChild.y − parent.y
paddingLeft   = firstChild.x − parent.x
paddingRight  = (parent.x + parent.width) − (lastChild.x + lastChild.width)
paddingBottom = (parent.y + parent.height) − (lastChild.y + lastChild.height)
```

**Header → body section gap (page shells):**

```
gap = bodyGroup.y − (header.y + header.height)
```

Skill: **`figma-page-shell-layout`**.

**Between unrelated groups:** measure from **bottom of group A** to **top of group B**, not from parent `y`.

### 2.1 Overlapping siblings (header.h vs content.y)

When a header frame reports `height: 72` but the next sibling Content has `y: 68` (4px overlap), **do not** implement `headerH + contentPaddingTop` blindly.

Truth for field placement is **absolute Y inside the card/frame**:

```
firstFieldAbsY = content.y + firstRow.y   // e.g. 68 + 20 = 88
formPaddingTop = firstFieldAbsY − header.height   // e.g. 88 − 72 = 16
```

Playwright gate (card-relative):

```
field[0].top − card.top === firstFieldAbsY   // Fail if Δ ≥ 2px
```

Common fail mode: `header 72` + `pt 20` → fields at **92** while Figma fields sit at **88** — every row and the notes block ghost by +4px in Diffchecker.

### 3. Classify the layout mode

| Metadata pattern                                 | CSS mapping                                      |
| ------------------------------------------------ | ------------------------------------------------ |
| Consistent derived gaps (±1px)                   | `Stack spacing` / `gap` = median gap             |
| Gaps increase toward edges, no fixed itemSpacing | `justifyContent: 'space-between'` — **no** `gap` |
| Mixed row + column                               | Nested `Stack` / MUI `Grid` — one axis per frame |
| Overlapping boxes                                | Absolute positioning — do not force flex         |
| Only one child                                   | Padding only; gap N/A                            |

### 4. Cross-check MCP `get_design_context`

- If MCP `gap` **matches** median derived gap (±1px) → use it.
- If MCP `gap` **contradicts** metadata or Dev Mode omits gap → **metadata wins**; note in report.
- If MCP returns `gap: 0` but derived gaps are 16/24 → use derived values.

### 5. Map px → theme

Repo scale: **`theme.spacing(1) = 4px`** (`spacing.ts`).

```
Stack spacing={6}  → 24px   (not MUI default 8px grid)
sx gap: '24px'     → when Figma says 24 exactly
```

Prefer `Stack spacing={n}` when gap equals `n × 4`. Use raw px in `sx` only when Figma is off-scale (e.g. 84.5) or documented one-off.

### 6. Runtime verify (Playwright)

After implementing, load **`playwright-verify-fix-loop`**:

1. Navigate + viewport match Figma frame (default agent **1530×920** unless frame differs)
2. `browser_snapshot` with **`boxes: true`**
3. Match snapshot `[box=x,y,w,h]` to metadata ids by role/text/order
4. `browser_evaluate` / `browser_run_code` for `getBoundingClientRect` on key nodes
5. Table: **Figma derived | MCP claimed | Runtime measured | Δ | Fix**

Tolerance: **≤2px** subpixel OK; **≥4px** = fail → fix owning component.

## Report template

```markdown
### Spacing audit — [frame name] ([nodeId])

| Relationship   | Figma derived | MCP gap/padding | Runtime (Playwright) | Status                    |
| -------------- | ------------- | --------------- | -------------------- | ------------------------- |
| header → cards | 84.5px        | 84.5 / none     | 85px                 | Pass                      |
| card gap       | 24px          | 18px (computed) | 24px                 | MCP wrong — used metadata |

**Layout mode:** column flex + gap 24  
**Suspicious MCP fields ignored:** itemSpacing (SPACE_BETWEEN)
```

## Checklist

- [ ] `get_metadata` on parent **and** spacing-owning children
- [ ] Derived gaps from sibling coordinates, not guessed Stack spacing
- [ ] SPACE_BETWEEN / gap Auto → `justifyContent`, not fake `gap`
- [ ] Page inset vs header-only padding distinguished
- [ ] Playwright boxes confirm after fix
- [ ] User told when MCP gap was rejected

## Mistakes

- Using MCP `gap` when Dev Mode shows no gap property
- Using `Stack spacing={6}` as default without measuring Figma
- Measuring from parent `y` instead of previous sibling bottom
- Ignoring nested frame padding (double-counting inset)
- Skipping runtime verify after metadata math
- Treating slot wrapper as having no flex when parent metadata shows column of children

## Reference

- My Card Scan desktop frame `25133:464645` — page inset 24; header→cards 84.5; card gap 24
- Figma forum: slots may omit auto-layout in MCP output — read parent frame metadata
