---
name: module-federation-safe-editing
description: Safely edit routing, module loading, and federation wiring. Use when touching module-federation.config.ts, rspack.config.ts, moduleLoader, or portal/ecosystem/zone remotes.
---

# Module Federation Safe Editing

Three layers: `apps/portal` (host) → `libraries/ecosystems/*` (remotes) → `libraries/zones/*` (nested remotes).

## Modes

| Command                             | Mode                                          |
| ----------------------------------- | --------------------------------------------- |
| `pnpm nx serve portal` / `pnpm dev` | Vite + path-alias `moduleLoader` (default DX) |
| `pnpm nx serve-mf portal`           | Rspack MF local smoke                         |
| `pnpm nx build portal`              | Rspack MF production                          |

Do **not** restore Vite Module Federation (`@module-federation/vite`, `VITE_MODULE_FEDERATION_ENABLED`, `tsconfig.base.federation.json`).

## Before Editing

1. Which layer owns the route/feature?
2. Trust **code/config** over stale prose
3. Update **both** loaders when adding remotes: `moduleLoader.alias.tsx` + `moduleLoader.federation.tsx`

## Files To Check Together

- `module-federation.config.ts` (relevant layer)
- `rspack.config.ts` / `tools/rspack/createIceRspackConfig.ts`
- `apps/portal/vite.config.ts` (Vite-dev alias remap)
- `module-federation.ports.ts` / `module-federation.shared.ts`
- `libraries/shared/moduleLoader/*` (**required** load surface)
- ecosystems/zone route files
- `apps/portal/src/app/app.tsx`
- `tsconfig.base.json` (aliases required for Vite-dev)

## Contract

- MF container `name` === Nx project name
- Federation loader: `loadRemote(\`${project}/${Logical}Remote\`)`in`moduleLoader.federation.tsx`
- Alias loader: `import('@ice-web-app/…')` in `moduleLoader.alias.tsx`
- Vite remaps `@ice-web-app/moduleLoader` → `index.alias.ts`; Rspack uses default `index.ts` → federation
- Each remote: `targets.serve.options.port` matches `MF_PORTS` (for serve-mf / URL resolver)
- Every MF config: `shared: mfSharedPolicy` + `additionalShared: mfAdditionalShared`
- Shared React family: `eager: true`; never share `@mui/utils` / `react-scan` / `react-icons/*` / `react-dropzone`
- `mfWorkspaceShared` required for store/i18n/hooks singletons (Vite-dev hides split-instance bugs)
- Portal `baseHref` `/portal/` → `publicPath: '/portal/'`

## Symptom → cause

| Symptom                                     | Likely cause                                               |
| ------------------------------------------- | ---------------------------------------------------------- |
| Header zone icons empty (MF)                | Missing `@ice-web-app/shared-store` in `mfWorkspaceShared` |
| Sidebar / auth diverge (MF)                 | Split Zustand / i18n across containers                     |
| `Tabs` / `unstable_getNormalizedScrollLeft` | `@mui/utils` singleton-shared                              |
| Vite-dev still hits remotes / loadRemote    | Missing Vite resolve alias to `index.alias.ts`             |
| Raw i18n keys in sidebar after build        | `loadModuleNamespace` not at module scope                  |

## `MF_PRODUCTION`

- **false / unset:** remotes = project names; Continuous Tasks on `serve` (remotes) when using `serve-mf`
- **true:** absolute `MF_*_URL` (Node/build-time)

## Vite that remains (intentional)

- Portal **dev serve** (`apps/portal/vite.config.ts`)
- Vitest runner
- `designSystem` library build (`@nx/vite/plugin` include)

## Mistakes

- Dropping `moduleLoader` or updating only one of alias/federation loaders
- Sharing `@mui/utils` as a singleton
- Reintroducing Portal `./Shared*` exposes, `*.federation.ts` shims, Vite MF, or `tsconfig.base.federation.json`
- Using Vite `build` as the production portal artifact
- Missing remote `serve.options.port` (MF URL resolver → all remotes on 4201)
- Putting `MF_*` into browser `getEnvVariable` without intent
