---
name: figma-page-shell-layout
description: Figma page shells with PageHeader/back+breadcrumbs, contentPadding, fixed-width centered card rows, responsive gaps, and mobile overflow. Use when implementing a full page from Figma, sub-pages with Back + breadcrumbs, or card clusters with fixed px widths.
---

# Figma Page Shell Layout

Companion skills: **`figma-design-system-parity`**, **`figma-pixel-parity-check`**, **`mui-layout-grid`**, **`hide-main-sidebar`**, **`figma-responsive-ownership`** (desktop-only Figma → full responsive matrix).  
Companion skills: `figma-feature-delivery`, `figma-responsive-ownership`, `figma-design-system-parity`.
When multiple viewport frames exist, follow the stage order in `fullstack-feature-delivery-loop`.

## When

- New Figma **page / sub-page** (Back + breadcrumbs + body)
- Card clusters with **fixed Figma widths** (e.g. 361px) that must stay centered
- User complains about too much/little padding around `PageHeader`
- Gaps between header and body that are **not** `Stack spacing={6}` (e.g. 84.5px)
- Mobile: right padding “gone”, fixed cards overflowing, or header→body gap too large

## Portal content padding vs Figma frame inset

`LayoutContent` default padding is roughly `10–30px` by breakpoint. Override when Figma owns the edges:

```tsx
const { setContentPadding } = useGlobalStore();

useEffect(() => {
  setContentPadding(0);
  return () => setContentPadding(undefined);
}, [setContentPadding]);
```

### Frame inset is usually PAGE padding — not header-only

Read the **Main frame** and the **first content group** positions from `get_metadata`:

| Node         | Example (My Card Scan `25133:464645`) | Meaning                            |
| ------------ | ------------------------------------- | ---------------------------------- |
| Header group | `x=24`, `y=24`, `h=44`                | Starts at frame inset              |
| Cards group  | `x=24`, `y=152.5`, `h=520`            | Same left inset; body below header |

If header **and** body share the same `x` (e.g. both `24`), that inset is **page padding on the whole shell**, not a wrapper around `PageHeader` alone.

```tsx
// ✅ GOOD — Figma frame inset on the whole page
<Stack sx={{ p: '24px', boxSizing: 'border-box' }} spacing={0}>
  <PageHeader backButton breadCrumbs={…} />
  <Stack sx={{ mt: { xs: '24px', md: '84.5px' }, … }}>{/* body */}</Stack>
</Stack>

// ❌ BAD — 24px only on the header
<Box sx={{ pt: '24px', pl: '24px' }}>
  <PageHeader … />
</Box>
```

| ❌ Mistake                                                     | Why it fails                                         |
| -------------------------------------------------------------- | ---------------------------------------------------- |
| Header-only `pt/pl: 24` when body also sits at `x=24`          | Cards/body lose the shared frame inset               |
| Page `Stack` with `px={6} py={6}` **on top of** portal padding | Double padding — always `setContentPadding(0)` first |
| Inventing `px={{ sm: 6 }}` without reading metadata `x`/`y`    | Drift from Figma                                     |

### Header → body gap (desktop math + responsive taper)

Desktop (from metadata):

```
gap = cards.y − (header.y + header.height)
```

**PageHeader empty-stack trap (Account Settings `6030:211369`):** when `title` sits beside `Back` (no breadcrumbs / no description), `PageHeader` must **not** mount the second inner `Stack`. Parent `spacing={4}` still inserts **16px** between the Back row and an empty sibling → measured `headerBottomToCard` becomes **40** instead of Figma **24** (`page Stack gap={6}` = 24 + ghost 16). Gate: `card.top − back.bottom === Figma gap` (±1) **and** PageHeader outer height ≈ Back row height (44), not 60.

**My Memberships (`10992:327655`):** Figma Main header includes **Back** + title — use `PageHeader … backButton={{ size: 'medium' }}` (same as Account Settings). Card body pad **20** only (`padding: 5`, no extra `px`); Card frame is **1222×710** even when content is shorter → `minHeight: 710`.

When **only a desktop frame** exists, still make the gap responsive — a stacked mobile column with 84.5px between header and first card is usually too tall:

```tsx
mt: { xs: '24px', sm: '40px', md: '84.5px' }
```

If MCP returns a `gap` that **contradicts** metadata or Dev Mode shows no gap → skill **`figma-implicit-spacing`** (coordinate math wins).

If a mobile Figma frame exists, measure that frame’s gap the same way and use those exact values at the matching breakpoint.

| ❌ Mistake                                                   | Why it fails                       |
| ------------------------------------------------------------ | ---------------------------------- |
| Hardcoding only `mt: '84.5px'` for all breakpoints           | Mobile looks sparse / oversized    |
| Using `cards.y` as `mt`                                      | Includes top inset + header height |
| Changing page padding and **not** re-validating desktop `mt` | Gap silently doubles or collapses  |

## Fixed-width cards — desktop lock, mobile fluid

Desktop Figma width (e.g. **361**) is a **`maxWidth`**, not a forever `minWidth`.

**Overflow trap (iPhone SE 375):** `p: 24` × 2 = 48 → content = 327. A card with `minWidth: 361` overflows and **eats the right padding** (left still looks padded).

```tsx
const CARD_W = 361;

// ✅ Page row
<Box width={{ xs: '100%', md: CARD_W }} maxWidth={CARD_W} minWidth={0}>
  <ScanFeatureCard />
</Box>

// ✅ Card shell
sx={{
  width: '100%',
  maxWidth: CARD_W,
  minWidth: 0, // never minWidth: CARD_W on mobile
  boxSizing: 'border-box',
}}
```

| ❌ Mistake                                            | Why it fails                                   |
| ----------------------------------------------------- | ---------------------------------------------- |
| `width/minWidth/maxWidth: 361` on every breakpoint    | Mobile overflow; right page padding disappears |
| `flexShrink: 0` + fixed 361 inside padded page        | Same overflow                                  |
| `Grid item md={4}` for fixed 361 cards                | Fluid columns stretch; gap ≠ Figma 24px        |
| CSS `display: 'grid'` / `gridTemplateColumns` in `sx` | Forbidden                                      |

Centered desktop cluster:

```tsx
<Stack
  direction={{ xs: 'column', md: 'row' }}
  justifyContent="center"
  alignItems={{ xs: 'center', md: 'stretch' }}
  flexWrap="wrap"
  width="100%"
  sx={{ gap: '24px', mt: { xs: '24px', sm: '40px', md: '84.5px' } }}
>
  …
</Stack>
```

**Sanity check:** `cardMaxWidth + 2 × pagePadding ≤ viewportWidth` at the smallest target phone (e.g. 375). If not, cards must be fluid.

## Edge-anchored actions (Save / Next / primary CTA)

Figma desktop frames are often a **fixed content size** (e.g. Main `1180×826` inside agent `1530×920`). Controls near the frame corner are usually **edge pins**, not flow margins that only look right at that height.

### Detect from metadata

```
rightInset  = frame.width  − (button.x + button.width)   // often 24
bottomInset = frame.height − (button.y + button.height)  // often 24
```

If `rightInset` / `bottomInset` equal the page inset (or are clearly “to the frame edge”), implement as **content-pane anchored**:

```tsx
// ✅ Fill portal main + pin CTA to content bottom/right (page p already = 24)
<Stack
  className={`${commonLayoutClasses.contentHeightFixed} flex is-full`}
  sx={{ p: '24px', flex: 1, minHeight: 0, height: '100%', display: 'flex', flexDirection: 'column' }}
>
  <PageHeader … />
  <Stack flex={1} minHeight={0} sx={{ mt: '24px', overflow: 'auto' }} alignItems="center">
    {/* body / card — do NOT justifyContent: 'center' unless Figma centers the block in leftover space */}
  </Stack>
  <Box sx={{ display: 'flex', justifyContent: { xs: 'center', md: 'flex-end' }, flexShrink: 0 }}>
    <DsButton>Save & Next</DsButton>
  </Box>
</Stack>

// ❌ BAD — mt after card that only equals Figma gap at one viewport height
<Stack>{card}</Stack>
<Box sx={{ mt: '28px', display: 'flex', justifyContent: 'flex-end' }}>
  <DsButton>Save & Next</DsButton>
</Box>
```

| ❌ Mistake                                                           | Why it fails                                                  |
| -------------------------------------------------------------------- | ------------------------------------------------------------- |
| `mt: 28` (or similar) under the card for a bottom-right Figma button | On taller screens the CTA floats mid-pane; gap to bottom ≫ 24 |
| Vertically centering the card in `flex: 1` leftover space            | Shifts card abs Y vs Figma; CTA still wrong if flow-based     |
| Pixel-diff / Playwright **only** at 1530×920                         | Misses edge-pin regressions on larger panes                   |

**Parity vs responsive:** geometry/pixel checks use the Figma/agent desktop size. Separately, smoke a **taller/wider** viewport and assert `contentBottom − buttonBottom ≈ pageEdge` and `contentRight − buttonRight ≈ pageEdge` (tolerance ≤ 2px).

Skill companions: **`figma-playwright-parity`**, **`playwright-verify-fix-loop`**.

## Loading / empty data

Do not render `` `${firstName} ${lastName}` `` → `"undefined undefined"`. Keep avatar + title on skeletons until names exist.

## Overlay drawers

Do not invent `right: 16` on temporary drawers unless Figma shows an inset.

## Avatar / image pickers

Prefer a skeleton tile grid matching the real layout over a centered spinner.

## Overhangs, exact gaps, Figma line breaks (scan cards)

Lessons from My Card / Scan hub (`25133:464645`) parity:

| ❌ Mistake                                                                    | Why it fails                                                                                                                            |
| ----------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `pt` / spacer to “make room” for an absolutely positioned avatar over a QR    | Increases header→QR gap (Figma 41px becomes 41+13); QR sits too low in slider diffs                                                     |
| `spacing={10}` when metadata gap is **41**                                    | 40px ≠ 41px — use `gap: '41px'` from coordinate math                                                                                    |
| Hand-approximated header/action SVGs                                          | Icon silhouette fails Diffchecker even when layout numbers match                                                                        |
| Ignoring Figma forced line breaks in body copy                                | Wrap/line count drifts; store `\n` in i18n + `whiteSpace: 'pre-line'`                                                                   |
| Stack `spacing` **plus** `MuiDialogActions` `marginInlineStart` on 2nd button | Gap doubles (~32px vs Figma 16px) — zero the theme margin on the actions row                                                            |
| Locking scan form cards to Figma **`maxWidth: 1132`** forever                 | Past 1530×920 the white card stays 1132 while the pane grows → side gutters; edges must stay at page `p: 24` (`width/maxWidth: '100%'`) |

```tsx
// ✅ Avatar overhangs without shifting layout
<Box position="relative" width={120} height={125.85} overflow="visible">
  <Box /* QR frame fills box */ />
  <DsAvatar sx={{ position: 'absolute', top: -12.95, left: '50%', transform: 'translateX(-50%)' }} />
</Box>

// ✅ Scan form cards (manual add / business review) — Figma 1132 = pane fill at 1180 Main (24+1132+24)
// Past agent 1530×920: grow with the content pane; do not center a fixed 1132.
<DsCard sx={{ width: '100%', maxWidth: '100%', … }} />
```

## Checklist

- [ ] `setContentPadding(0)` when Figma owns edges; restore on unmount
- [ ] Frame inset on the **whole page** when header + body share `x` — not header-only
- [ ] Desktop header→body gap = `cards.y − (header.y + header.h)`; **responsive** `mt` if no mobile frame
- [ ] Back+title-only PageHeader: no empty description stack (ghost `spacing={4}` → +16 gap)
- [ ] Cards: `maxWidth` = Figma width; `width: '100%'`, `minWidth: 0` below `md`
- [ ] **Exception — full-pane form cards** (scan manual / review `1132`): `width/maxWidth: '100%'` so >1530 does not grow side gutters
- [ ] Intra-card section gaps from metadata (e.g. **41px**), not nearest `spacing(n)` guess
- [ ] Absolutely positioned overhangs do **not** add sibling padding/margin
- [ ] Edge CTAs: pin to content pane when Figma inset-to-frame ≈ page padding — not one-shot `mt` under the card
- [ ] Verify phone viewport: page padding visible on **both** left and right
- [ ] Verify one **larger-than-Figma** viewport for edge-anchored footers
- [ ] Inform user of zone-local custom UI
- [ ] Fixed-height cards with inner scroll: scrollport is **inset** from the card edge (top/right/bottom)
      so the native scrollbar is not flush to the chrome; scrollbar chrome itself lives in
      `materio/globals.css` (`shared-component-guard`)

## Reference

- My Card Scan: `libraries/zones/collaborationZone/src/pages/iceContacts/myCard/scan/`
- Upload from Gallery edge CTA: `…/scan/business/upload/` — Figma `26487:469962` (Save & Next at frame − 24)
- Figma desktop: `25133:464645` — header `24,24,h44`; cards `24,152.5` → gap **84.5**; card width **361**; intra-card section gap **41**; share QR **120×125.85** + avatar top **-12.95**
