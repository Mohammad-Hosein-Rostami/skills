---
name: shared-library-discovery
description: Mandatory shared-first discovery before creating reusable UI, hooks, utils, state, services, or types. Use when adding anything that might belong in libraries/shared.
---

# Shared Library Discovery

**Shared-first is mandatory.** Run this before creating reusable code.

## Check Order

1. `libraries/shared/designSystem` — DS primitives
2. `libraries/shared/ui` — app-level composites
3. `libraries/shared/hooks`
4. `libraries/shared/utils`
5. `libraries/shared/store`
6. `libraries/shared/services` — REST/Hey API client (preferred); legacy GraphQL only where it already exists
7. `libraries/shared/types`
8. `libraries/shared/assets`

Use `@ice-web-app/*` aliases, not deep cross-library relative imports.

## Placement Rules

| Need                          | Likely home                                                |
| ----------------------------- | ---------------------------------------------------------- |
| Generic UI primitive          | `designSystem`                                             |
| Compact month calendar        | `designSystem` → `DsMiniCalendar` (`components/calendar/`) |
| Shared page-level composition | `shared/ui`                                                |
| Feature-specific logic/UI     | ecosystem or zone                                          |
| New API access                | REST via `useApi` / Hey API — not new GraphQL              |

## Zone-local exceptions

**ICE Calls / Jitsi** — runtime is intentionally **not** in shared packages:

- `useJitsiStore` + lib-jitsi-meet → `collaborationZone/.../pages/iceCalls/store/`
- Still shared-first for UI (`designSystem`, `shared-ui`), hooks (`useApi`), and REST — only the Jitsi/call runtime stays zone-local
- Workflow: `.cursor/skills/ice-calls-jitsi/SKILL.md` · `docs/ice-calls/README.md`

Do not move Jitsi runtime to `libraries/shared/store` without an explicit refactor task.

## DS Usage Sources

1. Package exports + component source in `designSystem` / `shared/ui`
2. Real usage in ecosystems and zones
3. Theme files: `libraries/shared/designSystem/src/materio/@core/theme/`

Storybook has been **removed** — do not add stories or treat Storybook as authoritative.

## Styling Default

MUI + `sx` + theme tokens. **No Tailwind** for normal feature work.

## Output

State one of:

- Reuse at `<path>`
- Extend at `<path>`
- No shared fit; local implementation justified

## Mistakes

- Custom modal/table/header before checking DS / `shared/ui`
- New global state before checking `shared/store`
- Reintroducing Storybook instead of using exports, source, or feature usage
- Tailwind for ordinary feature styling
