---
name: figma-absolute-layout-safety
description: Keep absolutely/fixed positioned UI (overhangs, badges, avatars on QR, floating CTAs, overlays) from overlapping or clipping wrong content across responsive breakpoints. Use when Figma has position absolute/fixed, negative offsets, or overlays.
---

# Absolute / Fixed Layout Safety (Responsive)

Companion: **`figma-page-shell-layout`** · **`figma-responsive-ownership`** · **`figma-card-relative-geometry`** · **`playwright-verify-fix-loop`**.  
Rules: `design-system-and-styling.mdc` (Overlays / centring / clipping) · `figma-parity-geometry-gates.mdc`.

## When

- Figma layers with absolute / fixed / negative `top`/`left` (avatar on QR, badges, scrubbers, floating actions)
- Overlays, viewfinder frames, warning chips on camera/QR viewports
- Anything that “sits on top of” another region without taking flow space
- Responsive bugs: absolute element covers text/buttons on phone, or leaves a hole because a spacer was invented

## Principles

1. **Flow owns layout; absolute owns decoration.** The parent’s in-flow size must match Figma’s reserved box. Absolute children must not be “made room for” with extra sibling `pt`/`mt` — that shifts everything else (classic QR avatar mistake).
2. **One centring method.** Do not combine flex `alignItems: 'center'` with `translateY(-50%)` on the same axis (double centre).
3. **Do not absolutely centre a variable-width pill inside a narrower fixed column** — it clips one side and overlaps the neighbour. Anchor an edge or widen the host.
4. **Check `overflow` ancestors.** `overflow: auto|hidden` clips absolute children; verify the chain before relying on overhang.
5. **Responsive:** absolute offsets that work at 1530×920 often collide when the host shrinks. At phone/tablet:
   - Prefer switching to in-flow stack when the host is too narrow
   - Or scale offsets with `%` / clamp within the host’s `position: relative` box
   - Never leave a fixed `left: 400px`-style desktop coordinate on mobile
6. **Composite markers** (label + connector + rule) stay **one flex row**, centred once — skill notes in design-system rules.

## Implementation pattern

```tsx
// ✅ Host reserves Figma size; overhang does not add flow padding
<Box position="relative" width={hostW} height={hostH} overflow="visible" minWidth={0}>
  <Box /* in-flow / fill content */ width="100%" height="100%" />
  <Box
    position="absolute"
    top={figmaTop} // may be negative
    left="50%"
    sx={{ transform: 'translateX(-50%)' }}
  >
    {/* badge / avatar */}
  </Box>
</Box>

// ❌ BAD — spacer to “clear” the absolute child
<Box pt="13px">
  <Box position="relative">…</Box>
</Box>
```

On small breakpoints, if Figma has no mobile frame:

```tsx
sx={{
  position: { xs: 'static', md: 'absolute' },
  // or stack the badge above on xs
}}
```

## Playwright asserts (required with responsive matrix)

At **375**, **768**, and **1530**:

1. Snapshot with boxes for absolute element + nearest content siblings
2. Fail if absolute box intersects interactive text/buttons that Figma keeps clear (unless Figma shows intentional overlap)
3. Fail if absolute child is clipped and Figma shows it overhanging
4. Fail if host width collapsed and absolute `left`/`top` still use desktop px outside the host

## Do not

- Invent layout padding for absolute overhangs
- Use desktop pixel `left`/`top` unchanged on `xs` when the host is fluid
- Hide collisions with `z-index` wars instead of reflowing
- Skip absolute elements in the responsive matrix (they break most often)

## Related

- Page shell overhangs section: **`figma-page-shell-layout`**
- Geometry tables: **`figma-card-relative-geometry`**
- Full responsive matrix: **`figma-responsive-ownership`**
