---
name: skill-first-governance
description: Repo-wide operational guardrails for architecture, i18n, styling, API, verification, and learning capture. Load for any medium or large feature task, and when deciding whether a change belongs in shared code.
---

# Skill-First Governance

Load this for any medium/large feature task. It is the baseline; the delivery stages live in
`fullstack-feature-delivery-loop`.

## Governance pillars

1. **Architecture ownership**
   - Respect app / ecosystem / zone / shared boundaries.
   - Reuse `libraries/shared/*` before local duplication (`shared-library-discovery`).
   - Prefer a zone/feature-local fix over editing shared or high-level components
     (`shared-component-guard`). When shared code must change, state the blast radius first.

2. **UI discipline**
   - Design-system components and theme tokens first.
   - No `fontSize` / `fontWeight` / `lineHeight` in feature `sx` — use typography variants
     (`ds-typography-first`).
   - Icons: `DsIcon customIcon={<Name />}` with `Name` from `libraries/shared/assets/icons`
     (`ds-icon-svg-assets`). The `IceIconsEnum` webfont path (`icon="KEY"`) is legacy — never add one,
     and migrate the ones you touch.
   - After every icon migration, mark the call site `✅ DONE` in `docs/icon-refactor-inventory.md`
     (and `docs/collab-zone-icon-refactor.md` for collab files) in the **same turn** — record the
     shipped component name, not the inventory “Proposed” PascalCase. Stale tracker rows are a
     process failure (`ds-icon-svg-assets` rule 13).
   - Before creating any icon, run the reuse gate `pnpm icons:match <figma-export.svg>`. Reuse is
     decided by geometry, never by name — `TRASH` is the existing `Delete`, while `INFO_CIRCLE` is
     **not** `Info`. **Reject** a reuse claim that cites no verdict or diff ratio.
   - **Reject** substituting a different Figma file’s glyph when the user linked a specific node
     (e.g. management-info phones for Collab Zone hub call-status icons). Paint-convert the linked
     paths; do not swap geometry (`ds-icon-svg-assets`, `figma-exact-not-similar`).
   - Icon colour comes from `DsIcon color="<token>.<shade>"`, so glyphs paint `currentColor`. A baked
     hex is allowed only for a two-tone semantic illustration that declares its tone in a registry;
     a baked neutral grey is always a bug. If Figma paints fill **and** stroke on a path, both must
     become `currentColor` — dropping stroke yields invisible/half-tinted glyphs on light bg.
     **Reject** a new glyph carrying a raw Figma grey or a fill-only conversion of a stroked export.
   - **Call-site size = measured glyph (outer vs leaf).** Do not inflate `DsIcon size` or the
     hit-target above the linked instance (no 24→28/`xl` or 24→32). Examples: **card-modal
     `ice-Close`** outer 24 / glyph **14** / `iceGray.700` — also override `IconButton` `minWidth`
     (theme `sizeSmall` = **40**, else 40×24 wide chip); **Add New `ice-Plus`** outer 20 / glyph
     **11** / `iceGray.700` — reject passing outer px as `size` when leaf is inset
     (`ds-icon-svg-assets` rule 14). **Reject** migrations that keep an oversized box without a
     Figma measurement citing that larger frame.
   - Avoid one-off patterns where DS or shared-ui already solves the need.

3. **i18n discipline**
   - No hardcoded user-facing copy.
   - Add missing keys at the correct namespace/layer.
   - Verify no raw translation keys render in the UI.
   - Mutation success/error toasts: **`toast-copy-standards`** (i18n-first; no API `message` fallback for routine success).
   - A JSON string that looks truncated is often a prefix completed by styled JSX. Do not add `{{var}}` to that key unless the call site interpolates (`t(..., { var })` or `Trans`) and the extra concatenated siblings are removed.

4. **API discipline**
   - REST/OpenAPI only. GraphQL/Apollo is removed and must not return.
   - The generated client is source of truth — regenerate, never hand-edit.
   - Revert machine-local URL overrides after codegen.
   - After regen, fix `libraries/shared/types/*` if `Schema*N` aliases renumbered (no `as`).
   - List UIs after mutation/OAuth: `invalidateApiQueries` (+ `flushSync`) before refetch.
   - Return short, user-safe error messages.
   - Response `message` fields (mock/production) follow the same copy rules as FE toasts — see **`toast-copy-standards`**.

5. **Verification discipline**
   - Headed flow loop plus focused type-check and lint for the touched scope.
   - 1530×920 pixel parity for Figma work.
   - Responsive: the measured matrix in `figma-responsive-ownership`. **Reject** any Pass
     claimed only from "no page H-scroll", "no overflow", or "looks fine".
   - Table/heatmap UIs require numeric `columnAlign` per viewport in the final summary.
   - New shared icon: `pnpm icons:audit` must still report zero `IDENTICAL`/`RESCALED` pairs and
     zero `off-token` paints, with every two-tone icon matching its declared registry tone.
   - Fix root causes, not symptoms.

6. **Learning discipline**
   - Promote a repeated fix into a skill update. Prefer improving a skill over leaving a
     one-off lesson only in chat.

## Stop-and-ask conditions

Ask one blocking clarification before implementing when any of these is missing:

- Figma links / node IDs for UI work
- The route or path to validate against
- A definition of done, when the request is ambiguous

Everything else: proceed on a stated assumption and surface it in the final summary.

## Skills maintenance

- `.cursor/skills/` is canonical. `.agents/skills/` is a generated mirror.
- After editing any skill, run `pnpm skills:sync`. A pre-commit check fails on drift.
- Keep `SKILL.md` operational; move long reference material into the skill's `references/`.
- Prefer improving a skill over writing a long free-form prompt.

## Execution companion skills

- `fullstack-feature-delivery-loop` — stage order and deliverable format
- `toast-copy-standards` — FE/BE toast and API `message` copy
- `ds-icon-svg-assets` — required for any icon work
- `figma-responsive-ownership` — required when UI layout or Figma delivery is involved
- `nx-ice-verify` / `eslint-and-verification` — verification commands for this monorepo
