---
name: fullstack-feature-delivery-loop
description: End-to-end delivery loop and required deliverable format for features spanning UI, API, Playwright verification, 1530x920 pixel parity, responsive matrix, and hardening. Use for any multi-stage feature, and to know what a finished deliverable must contain.
---

# Fullstack Feature Delivery Loop

Use this when a task spans UI + API + verification. Companion baseline: `skill-first-governance`
(guardrails).

## Required input

Gather these before starting. Missing one of the first four for UI work is a stop-and-ask:

| Input                | Notes                                  |
| -------------------- | -------------------------------------- |
| Goal                 | Measurable outcome, not a direction    |
| Section / route      | The path to validate against           |
| Figma URL + node IDs | Required for any UI work               |
| Definition of done   | Required when the request is ambiguous |
| API expectations     | Endpoints, behaviors, messages         |
| Notes/constraints    | Anything that narrows the solution     |
| Out of scope         | Explicit exclusions                    |

Mock API repo: `/Users/mohammad-hosein-rostami/Desktop/Desap/ice-mock-api`

## Stage order (strict)

0. **Orient** — Map ownership and constraints from the request and codebase before editing.

1. **Context + ownership**
   - Map code ownership and dependencies.
   - Confirm existing shared components/hooks/services before creating new ones.

2. **UI implementation**
   - Implement the exact target Figma frame — never a similar peer (`figma-exact-not-similar`).
   - Design-system components and theme tokens first; i18n for all user-facing copy.
   - Icons: `DsIcon customIcon={<Name />}` from `libraries/shared/assets/icons` — never a new
     `icon="ENUM_KEY"` font usage (`ds-icon-svg-assets`). Ask for the Figma link if the glyph is missing,
     then run the reuse gate `pnpm icons:match` before creating a file. **`size` / hit-target = measured
     Figma boxes (outer vs leaf)** — never inflate (24→28/`xl` or leftover 32 buttons). **Card-modal
     `ice-Close`:** hit-target 24, glyph `size={14}`, `iceGray.700`. **Add New `ice-Plus`:** outer 20,
     glyph `size={11}`, `iceGray.700` — never pass outer as `size`. After each
     migration, mark `✅ DONE` in
     `docs/icon-refactor-inventory.md` (+ `docs/collab-zone-icon-refactor.md` for collab) in the same
     turn — shipped component name, not Proposed PascalCase.

3. **API implementation**
   - Align with related sections in the mock API; keep seeded data consistent across endpoints.
   - Return short, user-safe response messages — same copy rules as FE toasts (`toast-copy-standards`).
   - Update the endpoint `schema.md` on every create/change.

4. **Codegen + FE wiring**
   - Regenerate the OpenAPI client/types locally.
   - Revert machine-local URL overrides after generation.
   - Wire the FE through typed clients/hooks, never ad hoc fetches.
   - Success toasts: i18n-first (`toast.success(t(...))`), not `response.message` fallback.

5. **Playwright flow loop (headed)**
   - Run the real user flow. Capture failure, fix the root cause, rerun until stable.
   - Cover success, validation, empty, and error scenarios, and the user-facing messages
     for each.

6. **UI parity loop (1530×920)**
   - Capture app and Figma screenshots; compare geometry, spacing, typography, icons, colors.
   - Fix one root cause at a time and re-diff.

7. **Responsive loop**
   - Load `figma-responsive-ownership` (required for UI).
   - 1530×920 parity is necessary and **never sufficient**.
   - Full matrix 375→2400 with screenshots **and** geometry `evaluate` at every width.
   - Per width: top viewport + full-page, plus middle/bottom or container scroll shots when
     the page or a scroll container needs them.
   - **Forbidden Pass reasons:** "no page H-scroll", "no overflow", "screenshots look fine".
   - Tables/heatmaps: fill the numeric verdict table (`columnAlign` maxLeftΔ / maxWidthΔ).
     Blank cells are a Fail.
   - Fail if `maxLeftΔ > 1` (jagged rails / heat offset). Fix root layout; re-verify 1530.

8. **Finalization**
   - Re-run the scenario checks; remove debug leftovers.
   - Run focused type-check and lint for the touched scope (`nx-ice-verify`).
   - Run toast grep exit criteria from `toast-copy-standards` when mutation feedback changed.
   - Update the skill for anything learned that should not recur.

## Required deliverable format

Report in this shape when the task is complete:

1. **Changed files** — path plus what changed and why
2. **API changes** — endpoints, schema updates, codegen status
3. **Verification status** — headed flow, 1530 parity, type-check, lint
4. **Responsive verdict table** — required whenever UI layout changed
5. **Open questions and assumptions** — stated explicitly in the summary
6. **Remaining risks** — including what deliberately did not get done

## Hard fail rules

- Claiming done without the headed flow loop.
- Claiming a responsive pass from "no page H-scroll" / "no overflow" alone.
- Table/heatmap UI without numeric `columnAlign` evidence per viewport.
- Missing top / full-page / required scroll screenshot coverage for any viewport.
- Claiming done without verification output (commands and results) in the summary.
