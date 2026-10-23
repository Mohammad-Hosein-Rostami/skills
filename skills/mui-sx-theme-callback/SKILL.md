---
name: mui-sx-theme-callback
description: Correct MUI sx theme callback usage — top-level (theme) => only; nested selectors must close over theme. Use when styling with theme.typography/palette inside sx, DateCalendar overrides, or debugging undefined theme in nested sx.
---

# MUI `sx` Theme Callback

Companion skill: `skill-first-governance` (styling and theme consistency).

## Rule

**Only the top-level `sx` value callback receives `theme`.** Nested object values that look like `({ theme }) => …` do **not** get theme injected — `theme` is `undefined` and spreading `theme.typography.*` crashes.

## Correct

```tsx
// Top-level function; nested selectors close over `theme`
sx={(theme) => ({
  color: theme.palette.iceGray[700],
  '& .MuiPickersCalendarHeader-label': {
    ...theme.typography.subtitle1,
    color: theme.palette.iceGray[700],
  },
})}
```

Or palette-only shorthand at any leaf that MUI supports:

```tsx
sx={{
  color: ({ palette }) => palette.iceGray[700],
  borderBottom: ({ palette }) => `1px solid ${palette.iceGray['opacity-24']}`,
}}
```

## Incorrect

```tsx
// ❌ Nested `({ theme })` is NOT a theme callback — theme is undefined
sx={{
  '& .MuiPickersCalendarHeader-label': ({ theme }) => ({
    ...theme.typography.subtitle1, // boom
  }),
}}
```

## Custom / Hyphenated Palette Keys

MUI's `bgcolor` shorthand resolves dot-notation **only for standard palette shapes**. Custom keys with hyphens (e.g. `'opacity-8'`, `'opacity-16'`) are **not** resolvable via string shorthand.

**Wrong** (silently produces no color):

```tsx
bgcolor: 'iceGray.opacity16'; // key doesn't exist — hyphen missing
bgcolor: ({ palette }) => palette.iceGray[50]; // sx callback on Box receives theme, not palette
```

**Correct**:

```tsx
sx={(theme) => ({ bgcolor: theme.palette.iceGray['opacity-8'] })}
```

### `iceGray` scale trap

- `iceGray[50]` = `#FFFFFF` (white) — the numeric key 50 is white in this palette.
- The Figma "gray-8" item background (`#F5F5F5`) is `theme.palette.iceGray['opacity-8']`.
- Always verify numeric vs. opacity keys before using `iceGray[N]`.

## Circles

When the design is a **circle** (selected day, avatar, fab), use:

```tsx
borderRadius: '50%';
```

Do **not** use `borderRadius: '500px'` (or other huge px) to fake a circle.

## Pixel radii (Figma stroke caps / pills)

MUI `sx` treats a **numeric** `borderRadius` as **theme shape units**, not CSS pixels:

```tsx
// ❌ borderRadius: 2 → theme.shape.borderRadius * 2 (often ~12px)
borderRadius: 2;

// ✅ literal Figma px for a 4×16 round-cap accent
borderRadius: '2px';
```

## Shadows / elevations from Figma

Figma **Light/Elevation/shadow-xs** (`#4242421A`, offset `0,2`, radius `4`) maps to the theme — **not** a pasted `rgba`:

```tsx
// ❌ Wrong — hardcodes Figma export color
boxShadow: '0px 2px 4px rgba(66, 66, 66, 0.1)';

// ✅ Correct — DS elevation token (customShadows.ts)
boxShadow: theme.customShadows.xs;
// or: boxShadow: 'var(--mui-customShadows-xs)'
```

Same rule for `sm` / `md` / `lg` / `xl` / `iceDefault`. Prefer `theme.customShadows.*` over inventing `0px … rgba(...)`.

If you truly need a one-off tinted shadow from a palette color:

```tsx
import { alpha } from '@mui/material/styles';
boxShadow: `0px 2px 4px ${alpha(theme.palette.iceGray[700] as string, 0.1)}`;
```

## Reference

- `DsMiniCalendar`: `libraries/shared/designSystem/src/components/calendar/DsMiniCalendar.tsx`
- Shadows: `libraries/shared/designSystem/src/materio/@core/theme/customShadows.ts`
