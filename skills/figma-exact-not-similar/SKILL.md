---
name: figma-exact-not-similar
description: Enforce exact Figma UI — never substitute a similar peer screen, never skip small details, never approximate icons/gaps/copy. Use when implementing from Figma, parity work, or when an agent might reuse nearby UI that is only partly alike.
---

# Exact Figma — Not Similar / Not Nearby

Companion: **`figma-design-system-parity`** · **`figma-parity-fast-path`** · **`missing-ds-icon-anchors`** · **`figma-interactive-state`** · **`ds-typography-first`**.  
Rules: `figma-pixel-parity.mdc` · `figma-implementation.mdc`.

## Failure class (real)

Agents often:

1. Open a **nearby** route/component that “looks like” the target
2. Reuse that JSX / spacing / icons / copy with light edits
3. Skip **small Figma details** (exact gap px, icon glyph, line break, stroke presence/absence, field height, avatar overhang)
4. Call it done because it “feels close”
5. Reuse a **list row** that drops columns for some `cardType`s → uneven dashed dividers across rows
6. Lock **tabs** to a fixed width → clip “Analytics” and show a scroll chevron for two tabs
7. `flex-grow` / `ml:auto` a filter toolbar → huge empty band mid-row unlike Figma’s left cluster
8. Absolutely position a count badge that **overlaps** names; stretch status chips inside text columns
9. Open a detail dialog that hits a **different** API than the list entity → empty skeleton / “No Set”
10. Screenshot only **grid @ 1530** and skip list, overlay, and the responsive matrix
11. Use **filled** status chips when Figma shows **tonal** (light fill + colored label)
12. Ship list rows **without** status badges / action buttons when Figma list frames have them
13. Lock card `height` + `overflow:hidden` so Decline/Accept (or other CTAs) are **clipped**
14. Keep fixed `362px` cards on large screens → large **empty right** instead of fluid Grid fill
15. Omit kind-specific tabs (e.g. My Card Holder **Report**) because another frame only shows 2 tabs
16. Use one filter chrome for all tabs (Timeline+Cards everywhere) when Report is search-only
17. Densify My Card Holder to **6-up** (`xl={2}` / `xxl={2}`) to fill empty space → contact counts overlap avatars — **stretch** cards, keep Figma column count
18. Guess Report heatmap buckets instead of reading Figma cell fill tokens (`error-24` / `warning-24` / `warning-8`)
19. Treat a Statistics frame that omits Report as authority to **delete** Report — ignore the missing tab on that frame; kind still owns Report
20. **Center** status chips on the card when Figma places them under the identity column (left-aligned with name)
21. Inflate Declined card height / use `space-between` so chips and buttons float away from Figma’s fixed 24px Data→action gap
22. Lock Report heatmap columns to fixed px so large panes leave **empty space after Total** — stretch **all** columns by Figma fraction instead
23. Collapse a **multi-step** Figma dialog chain (e.g. Accepted delete info → confirm → toast) into one generic `DsConfirmModal` / wrong button copy
24. Fire pending **Resend** immediately without the Figma confirm (`14745:436554`) / ignore "only once"
25. Wire pending **Cancel request** confirm with swapped CTAs (Figma left outline cancels; right Keep dismisses)
26. Reuse generic `CONFIRM_ACCEPT_CONTACT` / Cancel·OK for **Cards You Declined** Change — Figma `14745:406727` is Decline|Accept + "'{name}' request?"; toast `14745:407329` / `ADDED_TO_CARD_HOLDER`
27. **ICE Contacts Share** — open a "Do you want to share…" confirm before select-card / permissions (PDF/Figma: Share goes straight into the share flow)
28. Approximate **MI list rows** with `%` flex columns and content-fit buttons — Figma locks role **200**/org **130**, name→status gap **16**, Cancel **144**/Resend **100**, Decline/Accept **100**, Change/Delete **153**
29. **ICE Contacts permissions drawer** — label footer `Save Changes` when Figma/PDF require **Confirm**
30. **ICE Contacts description** — render "Share Your Card" as plain body2; Figma uses **body2_500** emphasis on that phrase
31. **ICE Contacts list** — omit Role column for PERSONAL rows → dashed dividers misalign; always reserve Role (`"-"` via `displayOrDash` if empty)
32. **ICE Contacts pagination** — keep a broken Load More; Figma scrolls the board — use infinite scroll; never bump `useApi` `page` for load-more (remounts loading UI)
33. **ICE Contacts filters** — send `microsystemId` / `userType` when OpenAPI expects `microsystem` / `contactType`
34. **ICE Contacts list toggle** — use generic `BULLETED_LIST` instead of Figma ice-List SVG (`ANCHOR: missing-ds-icon/LIST_VIEW`)
35. **ICE Contacts filters** — `ml: auto` / `space-between` on the search → huge empty mid-row; Figma Filter Box is a **left cluster** (3×265 selects + 248 search, gap 16). Also: do **not** keep fixed `265`/`248` + viewport `flexWrap` — mid widths orphan Search beside Contacts (empty upper-right) or make Search narrower than the select above; size from the **filter card** with **container queries** (equal `calc(50% - 8px)` 2-up when content-box ≈**860–1079**px; Figma fixed + `nowrap` when content-box **≥1080**px — raised again from 720/1000 so wrap happens **sooner**; CQ is content-box so DsCard `p:5` means 1530 border≈1132 → content≈1092 still Figma row); always pass `SearchBox` `fullWidth` so default `maxWidth:248` does not inset the cell. Viewport `xl` (1536) is wrong for ICE 1530 desktop — sidebar changes content width independently of viewport. Shared by ICE Contacts / My Card Holder / New Card Requests.
36. **ICE Contacts PERSONAL** — seed `role: "Personal"` echoed in the Role column (and popup); Role slot shows `"-"`; Personal/Parent label only in company/status column (`t('PERSONAL')` / `t('PARENT')` + `cardTypeLabelColor`)
37. **ICE Contacts scan hub** — declare fail after 2s while MF/portal still shows "Loading, Please wait…"; wait until hub chrome (`Business Card` / `Share My QR Code`) is visible
38. **My Card Holder edit** — wire Edit to `POST /share-card`, open `EditCardDrawer` ("Edit My Card"), **or** keep `PermissionsDrawer` with footer **"Change Card"** when Figma is **31739:537377**. Correct UI is centered modal **977×636**: header **"Edit Your Permissions"** + back, left **Multi Card** carousel (chevrons), right `EditPermissionsForm` + footer **"Save Changes"**. APIs stay `GET …/permissions/{toUserId}` + `PUT …/permissions`; overlay pencil tooltip may still say Change Card. Wait for permissions GET before asserting Save enabled (form can paint from `theirCard` while `permissionCards` is still empty → Save disabled).
39. **New Card Request Decline confirm** — leave `DsConfirmModal` default `approveText={t('YES')}`; Figma `867:116377` primary is **Decline**
40. **New Card / Holder filters** — spread `microsystemId`/`userType` outside `query` (or wrong OpenAPI names) → empty UI while API has data; keep filters inside `useApi` `query` as `microsystem` + `contactType`
41. **Holder grid densify** — `xl={2}` / 6-up; Figma stays **4-up** (`lg/xl={3}`), cards **~265×228**
42. **Toast library mismatch** — use `react-toastify` directly in zone code when the app mounts shared toast (`@ice-web-app/shared-utils`) → silent success. Copy rules: **`toast-copy-standards`** (i18n-first, no `has been`, no routine `Successfully`)
43. **Data Contacts Individual Upload** — leave upload form **single-column** / wrong field order when Figma `42779:505447` is **2-col**: First|Last (*required) → Company|Role → Mobile|Email → Source|Location → DataContact|DataEntryMethod → Date full (calendar+**chevron**) → Notes; use `showRequiredAsterisk` on First/Last only (not every `required` field); `*` shows on **focus or value**
44. **DsDatePicker / DsTextField** — pass non-API `clearable` (leaks to DOM / fails types); use `hideClear` + optional `fieldEndAdornment="chevron"` for Figma date fields. Time-only dropdown is **not** a white centered pill list with a “Time” header — Figma `578:96935` is `primary.opacity-16` panel, left 16/22 rows h38, selected `opacity-32` + `primary.600`. **Toggle closed on second field click** like `DsSelect`: control `open`, toggle in `onInputClick`, **`preventOpenOnFocus`** (otherwise focus opens then the same click toggles shut), and while open `stopPropagation` on field `mousedown` (ignore-class is on TextField root, not the `<input>`). **Date calendar popper** = `DsFigmaMonthCalendar` (`25602:479207` = **265×284**, **5×7**): always **3** muted lead + trail fill (**1** for 31-day months). Same grid for `DsMiniCalendar`. Do **not** use react-datepicker/MUI Sunday-aligned weeks (August 2026 would show 6 lead / 5 trail). Class `ds-datepicker--figma-calendar` size/shadow overrides **after** base `.react-datepicker` styles. Type: month subtitle1 · weekday subtitle2 · day body2 · outside iceGray.300 · selected primary circle. Do **not** stretch to the Date field / inlineText trigger. Only **time-only** lists use `matchFieldWidthModifiers` / `ds-datepicker--match-field-width`.
45. **ICE Contacts detail overlay** — special-case only PERSONAL for green label and leave PARENT as `iceGray.500` / no purple ring; always use `cardTypeLabelColor` + `CirclePurpleBorder` / `CircleGreenBorder` for typed non-business overlays (cards **and** modals)
46. **Empty identity slots** — leave Role/Company/bio blank or `\u00A0` when Figma/product requires `"-"` via `displayOrDash` (grid, list columns, overlays, MI Contact Cards). **My Card Holder grid** PERSONAL/PARENT must still render **3 lines** (name / `"-"` / type label) — never only the type with a blank role slot. Treat legacy `"No Set"` / `NO_SET` as empty in `displayOrDash`.
47. **New Card Request Decline/Accept** — omit `color="primary"`, unequal widths, wrong radii, or list `flex:1` that stretches; Figma grid `9885:271494` = equal flex + gap 8 + h40 + Decline r6 / Accept r8; list `10171:294673` = fixed **100×40** shrink-0 + both r6; same pair in MI inbound actions
48. **Required label `*`** — gate on `required` alone or only on `hasValue`/`shouldShrink` (misses empty **focused**); use opt-in `showRequiredAsterisk` and render `*` from live `shrink` state (focus **or** value). Wire on Data Contacts upload/edit names + scan business add/review names
49. **Header ZoneMenu** — do not hide below `min-width:1380` only, or center with **content-only** `1fr auto 1fr` (looks right-shifted — logos live in the sidebar strip) or icon-only mid-width cluster that overlaps ActionMenu. **Desktop `≥1520`:** `position:fixed; left:50%; transform:translate(-50%,-50%); top:47` viewport-centers ZoneMenu (Figma full-bleed); reserve ~762 spacer in the content grid so ActionMenu cannot collide. Figma gap **16** / icon **56** / labels on. **Below 1520:** Zones **toggle** sits with **right ActionMenu** (not beside sidebar chevron); use **`DISTRICT_6_SHAPE`** hexagon — never `APPS` (that is the apps launcher). Same `Box` + **44×44** `CircleGoldBorder` chrome; labeled strip + `overflow-x: auto`, full-bleed past real pads. **Divider continuity:** NavHeader + StyledHeader both **94px border-box** + exact **`1px`** bottom border. Expand header only when zone bar open (`:has([data-test=zone-menu-bar-wrap])`); main row height **93** (=94−1). No backticks inside emotion template-literal comments (breaks Babel/HMR).
50. **MI pending Cancel Request wraps** — equal `flex:1` outline pair without `whiteSpace:'nowrap'` (and label) → "Cancel"/"Request" stacks inside h40; Figma `16030:540862` / `16035:380581` require **nowrap**; list Cancel **144**/Resend **100**; post-resend Resend stays mounted at **opacity 45%** (`16035:380583`), do not unmount
    50b. **Account Settings chrome / Change Image** — `PageHeader` with Back+title (no breadcrumbs/description) must not mount an empty second stack (`spacing={4}` ghost **+16** → header→card **40** vs Figma **24**); account nav first item must not stack `marginBlockStart:16` on top of nav `paddingBlockStart:16` (**32** vs **16**); `DsDropDownGroupButton` **small** = caption + `nowrap` (not body1) so "Change Image" stays one line in 172×40
51. **MI list vs ICE Contacts** — do not replace MI Figma role **200**/org **130** with ICE `contactListCols` flex truncation; inherit dash/type colors (`displayOrDash`, Personal/Parent role `"-"`) but keep MI column + button geometry from MI list frames
52. **Outlined input double left inset** — removing empty `startAdornment` (autofill wash / blue slab fix) drops `.MuiInputBase-adornedStart`; theme root `paddingLeft:16px` then **stacks** with MUI input `paddingLeft` → huge leading gap (Individual Upload Last Name/Email/Role). Keep skip-empty-adornment; zero horizontal padding on `.MuiOutlinedInput-input` so inset lives only on the root. Do **not** “fix” empty fields with `shrinkLabel` — that pins the label on the border forever and kills the MUI float; only use `shrinkLabel` when the field always shows a value (e.g. disabled Data Entry Method).
53. **My Card Holder Switch** — cycle active card **in-place** inside the 410 overlay when Figma `31586:450537` requires Switch → **"{Name}'s Cards"** modal **1172×636** (4× Multi Card **265×504**, gap **16**, header **68**). Edit on a Multi Card → existing **Edit Your Permissions** (`31739:537378`); hide cards modal while edit open; back restores cards modal. Seeds must put James Baker + David Knowles in holder relations even when they sit past `peers[:20]` alphabetically — force-include multi-card user IDs and share **all** their toCards via `GET …/details/{toUserId}`
54. **Edit Your Permissions Multi Card clipped** — leave carousel `Box` width 265 without `flexShrink:0` / `minWidth:265` → flex row with 40px chevrons squeezes card to ~257 and `overflow:hidden` clips actions/avatar. Figma frame **345×504** @ x8 = chevron40 + card265 + chevron40. Always paint chevrons (opacity when disabled). **Carousel data = their cards** from `GET …/details/{toUserId}` (Figma shows **David**, not login-user Ava from `GET …/permissions` / `my-card/{id}`). Chevrons **enabled when 2+ their cards**; start at the clicked card. Form toggles still come from `GET …/permissions` (my share flags). Multi Card actions = Bio + Edit + Trash (3). Share-mode `EditPermissionsForm` flush `p:0`; list **560**.
55. **Holder "{Name}'s Cards" modal** — Figma `31586:450538` is **1172×636** for 4-up (x32 pads + 4×265 + 3×16). PARENT Multi Cards must show **Contacts** count (do not gate on BUSINESS|PERSONAL only). Contacts color: BUSINESS turquoise / PERSONAL success / PARENT purple. Empty phone / legacy **"No Set"** → **"-"** via `RenderTextLink`. Always reserve phone + ICE Calls + website rows on Multi Card. Card boxes `flexShrink:0` + `minWidth:265`.
56. **ICE Calls hub status phones** — when the user links Collab Zone nodes (`7400:259218` incoming, outgoing Group `1010107935`, `7400:259234` missed), do **not** substitute management-info solid call SVGs or any “cleaner” lookalike. Exploded MCP paint is a conversion problem (`ds-icon-svg-assets` / color-conventions failure mode B), not permission to change geometry. Provenance JSDoc must cite the linked node / group id.
57. **Icon call-site size inflation** — migrating webfont → SVG and keeping `size="xl"` (28) or a 32×32
    `IconButton` around a Figma ice-* instance “so it looks big enough,” **or** setting `DsIcon size`
    to the **outer** instance when the leaf is smaller inside it (card-modal `ice-Close` outer 24 /
    leaf 14; My Card `ice-Plus` outer 20 / leaf 11). Measure outer vs leaf; never treat outer = glyph
    size (`ds-icon-svg-assets` rule 14).
58. **Account Connections “sync”** — invent an ICE 2-step provider form, embed a FE Google SDK, paste a
    Figma sample address (`setayesh…@gmail.com`), or `window.location.assign` the OAuth URL (full-page
    refresh). Correct: backend `oauth/start` + **popup** handshake; list refresh via
    `invalidateApiQueries` (+ `flushSync`) — see `figma-feature-delivery` / `rest-api-workflow`.
59. **Account Connections chrome drift** — skip selected-provider `primary.20`, put default email
    mid-list, draw dividers between every non-default row, or approximate New Email goals
    (`39314:460034`) with wrong width/chip variant / hide prior default when setting a new one.

That produces wrong UI. **Similar ≠ correct.**

## Hard rules

1. **Linked Figma frame is the only visual authority** for this task. Peer screens are reference for patterns (hooks, DS usage, routing) — not for layout, copy, icons, or micro-spacing. **Same for icons:** a linked glyph node wins over a similar icon from another Figma file or product.
2. **Before coding**, list the Figma node tree for the target frame (from `get_metadata`). Implement **that** tree — not the peer’s tree.
3. **Diff against Figma, not against the peer.** If peer has 3 cards and Figma has 2 panels + a CTA row, build Figma.
4. **Small details are mandatory**, not polish:
   - Exact gaps from metadata coordinate math (`figma-implicit-spacing`)
   - Exact icon SVGs (`missing-ds-icon-anchors` / `ds-icon-svg-assets`) — no “close enough” DS icon
   - Exact icon **size** — measure Figma outer vs leaf; never inflate for webfont optics. **Card-modal
     `ice-Close`:** hit-target 24×24, glyph `size={14}`, `color="iceGray.700"`. **Add New `ice-Plus`:**
     outer 20, glyph `size={11}`, `iceGray.700` — not outer as `size` (`ds-icon-svg-assets` rule 14).
   - Stroke presence **and** absence
   - Forced line breaks in copy (`\n` + `pre-line` when Figma wraps intentionally)
   - Field heights, radius, shadow tokens
   - Absolute overhangs without inventing spacer padding
5. **Interactive state from Figma tokens** (`figma-interactive-state`) — not from peer defaults.
6. **If a peer is used for wiring** (API hooks, form libs), strip its visual `sx`/structure and rebuild from Figma.
7. **Partial resemblance is a red flag** — stop and re-read Figma before continuing.
8. **List / table / grid are separate Figma frames** — do not ship only grid. Stable columns for every row; never omit dividers for PARENT/PERSONAL.
9. **Chrome that clips labels is a Fail** (tabs, breadcrumbs under overlays, truncated filter labels at desktop).
10. **Overlays must show the list entity** when the peer detail API returns empty for this section.

## Required checklist (per screen)

- [ ] Target Figma node-id recorded; screenshot reviewed
- [ ] Metadata abs positions / sizes captured for major regions
- [ ] Peer component (if any) used only for non-visual patterns; visual tree rebuilt from Figma
- [ ] Every visible Figma control has a counterpart (no silent omissions)
- [ ] No substituted icon / button variant / gap that “looks fine”
- [ ] Card-relative geometry Pass before claiming pixel parity (`figma-parity-fast-path`)
- [ ] Grid **and** list (if designed) measured; columns align across rows
- [ ] Detail overlay opened and populated (or documented empty-state from Figma)
- [ ] Tabs/filter labels fully visible at 1530; no spurious scroll chevrons
- [ ] Responsive matrix screenshots exist (not only 1530)

## Do not

- Copy `ScanFeatureCard` (or any peer) because the new frame is “also a card row”
- Keep peer padding/gaps when Figma numbers differ by even 1–4px
- Swap a missing Figma glyph for a generic `IceIconsEnum` that is wrong
- Skip footer/header stroke checks because “the peer has a divider”
- Implement 80% and leave micro-details for “later”
- Reuse `CardHolderListView` (or any peer list) when it collapses columns by type
- Lock two tabs into a width that clips “Analytics”
- Claim done from a single grid screenshot
- Show the Account Settings delete details textarea for every reason — it appears only when **Other** is selected, and Next stays disabled until that text is non-empty. Preset reasons send `details: ""`.
- Double-pad `DsConfirmModal` body with `px:5` when `MuiDialogContent` already uses spacing **5** (20px)
- Center the Account Deleted card with flex `center` — Figma sits ~**35px** above mid; title is **`h4_600`** not `h4` (500)
- Trust screenshot-caption AI for left-edge alignment — **measure** title vs row label `x` (e.g.
  Playwright/`getBoundingClientRect` or PNG column scan). Captions often swap which line is title vs content.
- When the user says collapse/accordion **content padding vs parent** is wrong: align nested labels with
  the **header title text** (same lead-in as the chevron hit-target column), not a second cell `pl` that
  leaves labels under the chevron while the title sits further right.
- Treat Figma sample emails / OAuth screenshots as real synced mailboxes on Account Connections
- Full-page navigate for provider OAuth when the page already owns the email-accounts list (use popup)
- Approximate New Email goals modal chips/width/prior-default visibility from a peer dialog

## Report

When finishing a UI phase, state explicitly:

```text
Exact-Figma compliance:
- Peer reused for: [wiring only | none]
- Visual source: Figma node [id] only
- Small details verified: gaps | icons | strokes | copy wraps | states
- Views verified: grid | list | overlay | responsive matrix
```
