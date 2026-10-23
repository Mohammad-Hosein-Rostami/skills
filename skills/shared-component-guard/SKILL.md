---
name: shared-component-guard
description: Prefer zone/feature-local fixes over editing shared or high-level components. When shared/designSystem/shared-ui changes are required, analyze blast radius, preserve defaults, and ask the user before risky changes.
---

# Shared Component Guard

**Use with** `shared-library-discovery`, `figma-design-system-parity`, and `design-system-and-styling.mdc`.

## Change preference order (strict)

1. **Feature-local** — `sx`, composition, zone utils (e.g. `iceCallsInCallMenus.ts`)
2. **DS props on instance** — `variant`, `size`, `color`, `sx` on existing export
3. **Extend shared primitive** — new optional prop, new variant, new token in theme (backward compatible)
4. **Change shared default behavior** — last resort; requires impact analysis + user approval

**High-level / widely used components** (`DsButton`, `DsDrawer`, `DsTypography`, `PageHeader`, `SearchBox`, `OptionMenu`, theme files) — steps 3–4 need extra care.

## Before editing `libraries/shared/*`

1. **Find all usages** — ripgrep importers and real feature usage
2. **Classify change**
   - **Additive** (new optional prop, new variant, new token) — usually safe if default unchanged
   - **Default change** (altering existing variant spacing/color) — **high risk**
   - **Breaking** (rename/remove prop, change default layout) — **stop and ask user**
3. **Figma justification** — document which frame requires the change and why instance-level `sx` is insufficient
4. **Blast radius table**

| File to change | Usages (count / key areas) | Risk         | Mitigation                     |
| -------------- | -------------------------- | ------------ | ------------------------------ |
|                |                            | low/med/high | opt-in prop, default preserved |

## When to ask the user (required)

Ask before applying if any of:

- Changing a **default** `variant`, `size`, color, padding, or radius on a shared component
- Editing `colorSchemes.ts` or `typography.ts` in a way that affects global theme
- Modifying `PageHeader`, `SearchBox`, `DsTable`, `DsForm` behavior used across ecosystems
- Grep shows **>3 unrelated feature areas** importing the component with the property you need to change
- Figma parity would be achieved by a breaking API change
- Change touches shared global behavior outside UI components (for example `useApi.utils`, global providers, route-level always-mounted wrappers)
- Root cause appears to be top-level/shared behavior and local fixes are likely to be fragile

**Ask format:** state file, proposed change, usages found, safer alternative tried, recommendation.

Also include:

- why local-only fixes are insufficient (if applicable)
- the lowest-risk top-level option
- blast-radius summary and rollback strategy

## Safe patterns

- New `DsTypography` variant — add to `typography.ts`; map in `DsTypography`; old variants unchanged
- New palette token — add step in `colorSchemes.ts`; use in feature via `palette.newToken`
- New optional `DsButton` prop with same default as today
- Zone util consolidating repeated in-call menu `sx` (ICE Calls pattern)

## Unsafe patterns (avoid)

- Tweaking `DsButton` `contained` padding globally for one Figma screen
- Changing `DsDrawer` default width for one zone
- Forking `PageHeader` into `iceCalls/PageHeader.tsx` when `rightElements` prop suffices
- `!important` in shared component to win a local fight

## Figma parity interaction

Pixel-perfect work **does not** justify breaking shared consumers.

1. Try feature-local + tokens first (`figma-design-system-parity`)
2. If shared extension needed → follow this skill
3. If user denies shared change → document Partial parity with local `sx` + comment

## Verify after shared edits

```bash
pnpm exec prettier --write [touched-shared-files] && pnpm exec eslint --fix [touched-shared-files]
pnpm nx run designSystem:type-check    # if designSystem touched
pnpm nx run ui:type-check              # Nx project name is `ui` (@ice-web-app/shared-ui)
pnpm nx run [affected-zone]:type-check
pnpm verify:affected                   # preferred agent PR loop
pnpm boundaries                        # if imports crossed layers
```

Spot-check 1–2 unrelated pages that import the changed component if defaults changed.

## Mistakes

- Editing `designSystem` first because it’s “cleaner”
- No usage grep before theme change
- Assuming Storybook exists or shows current DS behavior (Storybook is removed)
- Zone-only hack duplicated 5 times instead of one optional shared prop
- Silent default change that breaks collabGroups because ICE Calls needed 4px
- **Native scrollbar styling in `theme.config.ts` `MuiCssBaseline`** — that path does not win in
  practice. Edit **only** `libraries/shared/designSystem/src/materio/globals.css`. Do not dual-write
  scrollbar rules into `theme.config.ts`.
- **`scrollbar-color` / `scrollbar-width` alongside custom `::-webkit-scrollbar-track`** — in Chrome,
  setting the standard properties makes the browser **ignore** the webkit track/thumb rules, so the
  thumb paints thinner than the track. For a design rail + thumb that share one width, use **only**
  the `::-webkit-scrollbar*` rules (same `width` on the scrollbar, no thumb `border` inset).
