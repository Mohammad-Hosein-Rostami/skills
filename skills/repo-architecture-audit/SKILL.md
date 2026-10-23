---
name: repo-architecture-audit
description: Audit actual repo architecture before cross-cutting or multi-layer work. Use when docs may be stale or the task spans portal/ecosystem/zone/shared layers.
---

# Repo Architecture Audit

Run before structural, routing, federation, or cross-library changes.

## Read First (code > docs)

- `tsconfig.base.json`
- `**/module-federation.config.ts`, `**/rspack.config.ts`
- `module-federation.ports.ts`, `module-federation.shared.ts`
- relevant `project.json` (**tags**: `type:*` + `scope:*`)
- `apps/portal/src/app/app.tsx`
- `libraries/shared/moduleLoader/*`
- `nx.json`, `package.json`, `eslint.config.mjs` (boundaries only)

MF operator docs: `docs/module-federation-structure/`. Storybook has been removed.

Use Nx graph when unsure:

```bash
pnpm nx show projects -p 'tag:type:zone'
pnpm nx show project <name>
```

## Repo Model

| Layer             | Path                                    | Tags                             |
| ----------------- | --------------------------------------- | -------------------------------- |
| Host              | `apps/portal`                           | `type:app`, `scope:portal`       |
| Ecosystem remotes | `libraries/ecosystems/*`                | `type:ecosystem`, `scope:<name>` |
| Zone remotes      | `libraries/zones/*`                     | `type:zone`, `scope:<domain>`    |
| Shared Nx         | `designSystem`, `ui`                    | `type:shared`, `scope:shared`    |
| Shared aliases    | `libraries/shared/{hooks,services,...}` | **not** Nx projects              |

New packages: `@ice-web-app/workspace-plugin` generators (`zone` / `ecosystem` / `shared-lib`).

## Audit Questions

1. Which layer owns the change?
2. Which shared packages are involved — reuse before creating?
3. Hybrid modes: Vite-dev aliases vs Rspack MF prod — does any new path-alias singleton need `mfWorkspaceShared` for production?
4. New API work → REST (`rest-api-workflow`), not GraphQL
5. Generated files or project-specific commands involved?

## Output

- Current reality (from code/config)
- Affected layers and files
- Risks or doc/code contradictions
- Safest implementation path

## Policy Reminders

- Shared-first mandatory; `@ice-web-app/*` aliases
- MUI + `sx` + theme tokens — no Tailwind for normal features
- ESLint + Prettier for lint/format; no tests unless asked
- Regenerate `hey-api-client/**` via `pnpm openapi-codegen` — GraphQL removed
- Path-alias shared packages are **not** auto-federated; only `designSystem`/`ui` are Nx shared projects

## Mistakes

- Assuming alias-only Vite-dev means MF shares are unnecessary (prod still needs `mfWorkspaceShared`)
- Assuming every `libraries/shared/*` path is an Nx project (or is MF-shared automatically)
- Missing alias dependencies in `tsconfig.base.json` or missing `mfWorkspaceShared` for new singletons
- Choosing GraphQL for new work
- Assuming Module Federation is required for day-to-day `nx serve portal` (Vite aliases are default DX)
