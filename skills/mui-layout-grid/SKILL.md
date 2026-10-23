---
name: mui-layout-grid
description: Use MUI Grid container/item for multi-column layouts — never CSS grid via sx on Stack/Box. Use when building responsive columns, toolbars with left/center/right regions, card grids, or any layout that needs grid behavior.
---

# MUI Layout Grid

Companion skill: `skill-first-governance` (layout/UI discipline).

## When

- Multi-column / responsive column layouts
- Toolbars with left / center / right regions
- Card/list grids (contacts, calls, statistics cards)
- Any time you reach for `display: 'grid'` or `gridTemplateColumns` in `sx`

## Project convention

Import classic Grid from `@mui/material` (dominant pattern in zones):

```tsx
import { Grid, Stack } from '@mui/material';

<Grid container spacing={6} alignItems="center">
  <Grid item xs={12} md={4}>
    {/* left */}
  </Grid>
  <Grid item xs={12} md={4} display="flex" justifyContent="center">
    {/* center */}
  </Grid>
  <Grid item xs={12} md={4}>
    <Stack direction="row" justifyContent="flex-end">
      {/* right */}
    </Stack>
  </Grid>
</Grid>;
```

Reference usages:

- `libraries/zones/collaborationZone/src/pages/managementInfo/statistics/index.tsx`
- `libraries/zones/collaborationZone/src/pages/iceCalls/components/callScheduled/CallScheduledList.tsx`
- ICE Calls calendar toolbar (`pages/iceCalls/calendar/components/CalendarToolbar.tsx`)

## Prefer

| Need                                                   | Use                                                                                       |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| One axis spacing                                       | `Stack`                                                                                   |
| A 2-up / 3-up **form field row** inside a `Stack` form | `Stack direction={{ xs: 'column', md: 'row' }}` + `Box width={{ xs: '100%', md: '50%' }}` |
| Multi-column page/card **grid** with breakpoints       | `Grid` `container` + `item`                                                               |
| Nested row inside a cell                               | `Stack` inside `Grid item`                                                                |

## `Grid container` inside a `Stack` shifts the row by the gutter

Classic MUI `Grid` implements `spacing` with **negative margins on the container plus padding
on each item**. When the container is a flex child of a `Stack` (stretched to the cross axis),
that compensation does not land cleanly and the whole row ends up **inset by the gutter**
relative to its siblings.

```tsx
// ❌ BAD — this row sits ~16px right of the fields above and below it
<Stack spacing={5}>
  <DsTextField label="Meeting Title" fullWidth />
  <Grid container spacing={4}>
    <Grid item xs={12} md={6}><DsTextField label="Location" fullWidth /></Grid>
    <Grid item xs={12} md={6}><DsSelect label="Travel Time" fullWidth /></Grid>
  </Grid>
</Stack>

// ✅ GOOD — left edge aligns with every other field in the form
<Stack spacing={5}>
  <Box width="100%"><DsTextField label="Meeting Title" fullWidth /></Box>
  <Stack direction={{ xs: 'column', md: 'row' }} spacing={5} justifyContent="space-between">
    <Box width={{ xs: '100%', md: '50%' }}><DsTextField label="Location" fullWidth /></Box>
    <Box width={{ xs: '100%', md: '50%' }}><DsSelect label="Travel Time" fullWidth /></Box>
  </Stack>
</Stack>
```

A row of form fields is **one-axis spacing** — it does not need grid behavior. Reach for
`Grid` when you need real grid semantics (wrapping card decks, page columns, differing
per-breakpoint column counts), not for two inputs side by side.

## Mirror the sibling form before inventing a layout

When you add a second drawer/form to an existing feature family, copy the field-row structure
from the form that already ships. Hand-rolling a different one (`flex={1}` + `minWidth={0}`
where the sibling uses `width: '50%'`) produces rows that don't line up **between** the two
surfaces, even when each looks plausible alone.

Reference for ICE Calls drawers: `pages/iceCalls/components/scheduleMeeting/CreateScheduleForm.tsx`
— title row, date/time row, and two-column rows are all defined there.

## Do not

```tsx
// ❌ BAD — CSS grid via sx
<Stack sx={{ display: 'grid', gridTemplateColumns: { md: '1fr auto 1fr' } }} />

// ✅ GOOD — MUI Grid
<Grid container>
  <Grid item xs={12} md={4} />
  <Grid item xs={12} md={4} />
  <Grid item xs={12} md={4} />
</Grid>
```

```tsx
// ❌ BAD — Grid container as a flex child of a Stack (row inset by the gutter)
<Stack spacing={5}>
  <Grid container spacing={4}>…</Grid>
</Stack>

// ✅ GOOD — Stack row for a 2-up field row
<Stack direction={{ xs: 'column', md: 'row' }} spacing={5}>…</Stack>
```

Do not introduce `Unstable_Grid2` / `Grid2` unless the surrounding file already uses it.

## Exception: fixed-width Figma card clusters

When Figma specifies **fixed card widths** (e.g. 361px) and a **fixed gap** (e.g. 24px), centered as one group, do **not** use fluid `Grid item md={4}`. That stretches cells and breaks the gap.

Desktop width is a **`maxWidth`**. On mobile, cards must shrink (`width: '100%'`, `minWidth: 0`) or they overflow and eat **right** page padding (e.g. 375 − 48 = 327 < 361).

```tsx
// ✅ GOOD — maxWidth from Figma; fluid below md; exact gap; group centered
<Stack
  direction={{ xs: 'column', md: 'row' }}
  justifyContent="center"
  alignItems={{ xs: 'center', md: 'stretch' }}
  width="100%"
  sx={{ gap: '24px' }}
>
  <Box width={{ xs: '100%', md: 361 }} maxWidth={361} minWidth={0}>
    {/* card */}
  </Box>
  <Box width={{ xs: '100%', md: 361 }} maxWidth={361} minWidth={0}>
    {/* card */}
  </Box>
  <Box width={{ xs: '100%', md: 361 }} maxWidth={361} minWidth={0}>
    {/* card */}
  </Box>
</Stack>
```

```tsx
// ❌ BAD — locks 361 forever; overflows phones with page padding
<Box width={361} flexShrink={0} minWidth={361} />
```

Load **`figma-page-shell-layout`** for full-page Back/breadcrumb shells with this pattern (also covers responsive header→body gaps).

## Responsive verification (mandatory with `figma-responsive-ownership`)

Implementing a grid that only looks correct at **1530×920** is incomplete.

For every multi-column / card cluster / toolbar grid:

1. Define explicit breakpoint props (`xs` / `sm` / `md` / …) — do not rely on desktop defaults alone.
2. Headed-check at least **375**, **768**, **899** (just below `md`), **1530**, **1920**.
3. Fail if any viewport shows: horizontal scroll, fractional cards, overlapping cells, or actions unreachable.
4. Fixed Figma card widths → Stack + `maxWidth` path above; fluid content decks → `Grid` with `xs={12}` stacking.
5. After changing breakpoints, re-check **1530** so desktop parity did not regress.

Companion: **`figma-responsive-ownership`** · **`figma-absolute-layout-safety`**.
