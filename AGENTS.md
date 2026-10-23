# Agent Guide

Nx monorepo (React + TypeScript): `apps/portal` host, `libraries/ecosystems/*`,
`libraries/zones/*`, `libraries/shared/*`.

## Operating model

- **`.cursor/skills/`** — canonical, reusable procedures for how to do things.
  `.agents/skills/` is a generated mirror; never hand-edit it.

Start from these skills:

1. `skill-first-governance` — repo-wide guardrails
2. `fullstack-feature-delivery-loop` — stage order and deliverable format
3. `toast-copy-standards` — mutation toast copy (FE i18n-first + API `message` alignment)
4. `ds-icon-svg-assets` — icons via `DsIcon customIcon` + `libraries/shared/assets/icons`

## Core constraints

1. Shared-first reuse is mandatory (`libraries/shared/*` before local duplication).
2. REST/OpenAPI only for data access (no GraphQL/Apollo reintroduction).
3. i18n for all user-facing copy (no hardcoded strings); mutation toasts follow `toast-copy-standards`.
   ICE Calls recording started vs saved are **two** Figma chromes (`RecordingCallToast` variants) —
   never one shared class/icon for both (`ice-calls-jitsi`).
   Account Settings delete is **two-step** (reason → confirm) then public `/account/deleted`; mock
   soft-deletes only (never remove `user_me` seeds) — see `figma-feature-delivery` /
   `mock-api-openapi-sync`.
   Account Connections email sync is **backend OAuth** (popup + `oauth/start`; no FE Google SDK, no
   Figma sample addresses); after success invalidate the email-accounts GET cache before refetch
   (`rest-api-workflow` / `figma-feature-delivery`).
4. Generated clients are regenerated, not hand-edited. Mock: `pnpm openapi-codegen` →
   `hey-api-client`. ICE API: `pnpm openapi-codegen:ice` → `ice-api-client` (do not label it
   `real`; do not `export *` it). After OpenAPI regen, update `libraries/shared/types/*` if Hey
   API renumbered `Schema*N` aliases (TS2724) — never `as`.
5. Keep UI consistent with the design system and theme tokens. Native scrollbar chrome
   (track/thumb) lives only in `libraries/shared/designSystem/src/materio/globals.css` — do not
   restyle it via `theme.config.ts` `MuiCssBaseline`, and do not set `scrollbar-color` alongside
   custom `::-webkit-scrollbar-track` (Chrome then ignores the webkit rules).
6. Icons are shared SVG components: `DsIcon customIcon={<Name />}` from
   `libraries/shared/assets/icons`. The `IceIconsEnum` webfont (`icon="KEY"`) is legacy — never add a
   new one. Before creating an icon, prove it does not exist with `pnpm icons:match <export.svg>` —
   reuse is decided by geometry, not by name. Trackers: `docs/icon-refactor-inventory.md` (repo-wide)
   and `docs/collab-zone-icon-refactor.md` (collab focus). **Same turn as the code change:** mark
   each migrated call site `✅ DONE` with the **shipped** component (`TRASH` → `Delete`, social URL →
   `IceLink`, etc.) and remove it from Unresolved lists — do not leave pending rows after a refactor
   (`ds-icon-svg-assets` rule 13).
   **Linked Figma node wins:** never replace a user-linked Collab Zone glyph with a lookalike from
   another file (e.g. management-info solid phones for hub `CallIncoming` / `CallOutgoing` /
   `CallNoAnswered`). Fix paint on the **same** paths; do not swap geometry.
7. Icons paint with `currentColor` and are tinted by `DsIcon color="<token>.<shade>"`. A baked hex is
   allowed **only** for a two-tone semantic illustration, which must bake one tone's strong + `100`
   pair and declare that tone in a registry. Neutral grey is never a reason to bake a hex. If Figma
   sets both fill and stroke on a **clean** outline path, keep **both** as `currentColor` — dropping
   stroke leaves a vanishing hairline on light backgrounds. Never ship `stroke="currentColor"`
   without Figma's `strokeWidth` (bare stroke defaults to 1 and shreds exploded exports at 16px).
   Icons inside contained CTAs need an explicit `color` (e.g. `iceGray.50` on primary buttons) or
   `sx={{ color: 'inherit !important' }}`. See `ds-icon-svg-assets/references/color-conventions.md`.
   **Call-site size = measured glyph — outer vs leaf.** Never inflate past Figma (no `xl`/28 or leftover
   32 hit-targets). Examples: card-modal **`ice-Close`** hit-target 24 / glyph **`size={14}`** /
   `iceGray.700` — and **override `IconButton` `minWidth`** (theme `sizeSmall` is 40, else you get a
   40×24 wide chip); My Card **Add New `ice-Plus`** outer 20 / glyph **`size={11}`** / `iceGray.700` —
   never pass the outer px as `size` when the leaf is inset; Edit/Add card **“Add another” Text
   Button `ice-Plus`** (`18302:510070` / `18302:507392`) outer 14 / glyph **`size={10}`** /
   `primary.500` — never `size={14}` (`ds-icon-svg-assets` rule 14).
   **Collab social brand SVGs:** circular Size=L badges (`ice-Instagram` gradient, `ice-X` dark
   circle + white X, circular LI/YT/…) from Figma `10384:304918` / `33284:438*` — never outline IG /
   bare black X / square LI over those files.
8. Validate with focused type-check + lint + headed flow checks when behavior changes.
9. 1530×920 parity is necessary and never sufficient — responsive claims need the measured
   matrix from `figma-responsive-ownership`.
10. Behaviour changes update the matching docs in the same change (ICE Calls: `docs/ice-calls/`).

## Common commands

| Task              | Command                            |
| ----------------- | ---------------------------------- |
| Setup             | `pnpm run bootstrap`               |
| Serve portal      | `pnpm nx serve portal`             |
| Lint              | `pnpm lint` / `pnpm lint:staged`   |
| Format            | `pnpm format`                      |
| Type-check        | `pnpm nx run <project>:type-check` |
| Boundaries        | `pnpm boundaries`                  |
| Icon reuse gate   | `pnpm icons:match <export.svg>`    |
| Icon set audit    | `pnpm icons:audit`                 |
| Agent e2e smoke   | `pnpm e2e:agent:eyes-smoke`        |
| Sync skill mirror | `pnpm skills:sync`                 |

## Where to look

- Skills: `.cursor/skills/` (canonical), `.agents/skills/` (generated)
- Icon catalog: `.cursor/skills/ds-icon-svg-assets/references/icon-catalog.md` (125 icons)
- Icon colour rules: `.cursor/skills/ds-icon-svg-assets/references/color-conventions.md`
- Docs index: `docs/README.md`
- ICE Calls docs: `docs/ice-calls/README.md`
- Codex config: `.codex/config.toml` · Cursor MCP: `.cursor/mcp.json`
- Nx skills lock: `skills-lock.json`
