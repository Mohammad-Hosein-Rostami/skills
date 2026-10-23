---
name: eslint-and-verification
description: Verify changes with ESLint + Prettier and project type-check. Use after edits — not for adding tests or Storybook. Replaces biome-and-verification.
---

# ESLint And Verification

## Tooling Truth

- **ESLint** = lint (flat config, syntax-level; no type-aware projectService)
- **Prettier** = format
- **Boundaries** = `pnpm boundaries` (Nx graph). ESLint `@nx/enforce-module-boundaries` is off (crashes on `.tsx` barrels)
- **Tests:** do not add unless explicitly requested
- **Storybook:** removed — do not reintroduce
- CLI fails on **errors** only; warn-level rules print but do not fail (former Biome exit parity)

## Commands

| Task             | Command                                                       |
| ---------------- | ------------------------------------------------------------- |
| Lint             | `pnpm lint` / `pnpm lint:fix` / `pnpm lint:staged`            |
| Format           | `pnpm format` / `pnpm format:check`                           |
| Types            | `pnpm nx run <project>:type-check` / `pnpm type-check:staged` |
| Agent PR loop    | `pnpm verify:affected` (skill: `nx-ice-verify`)               |
| Layer boundaries | `pnpm boundaries`                                             |

Pre-commit: lint-staged (Prettier write + ESLint --fix on staged files).

## Pick The Smallest Check

| Edit type           | Verify with                               |
| ------------------- | ----------------------------------------- |
| Style/format        | Prettier on touched files                 |
| Lint                | ESLint on touched paths / project `:lint` |
| Types/logic         | project `type-check`                      |
| Cross-layer imports | `pnpm boundaries`                         |
| Behavior            | manual reasoning + lint/type-check/build  |

## Mistakes

- Using Biome or Prettier-via-ESLint (`eslint-plugin-prettier`)
- Enabling ESLint `fixAll` on save (Ctrl+S lag)
- Adding tests because tooling exists
- Enabling type-aware lint globally for speed-sensitive editor workflows
- Re-enabling `@nx/enforce-module-boundaries` without fixing `.tsx` barrel autofix crashes
- **Treating “project type-check failed elsewhere” as “our files are clean”** — grepping a red
  `tsc` log for touched paths and finding no matches is **not** a pass. Always run
  `eslint --max-warnings=0` on the touch set + scoped TS diagnostics (or ReadLints) on each
  touched file. See `ai/prompts/06-type-check.md`.
- **`catch (error)` while `const { error } = useApi(...)` is in scope** — shadows the query error;
  use `catch (err)` (IDE often flags this even when project `tsc` is noisy).
- **Scoped `tsc` without `strictNullChecks: true`** — portal `tsconfig` may omit `strict`, so
  `pnpm nx run portal:type-check` misses IDE errors like TS18048 (`'response' is possibly
'undefined'` from `useApi().execute()`). Always run touched-file diagnostics with
  `strictNullChecks: true` (see `ai/prompts/06-type-check.md`). Guard `execute()` results with
  `if (!response?.data) return` — do not `as` cast.
- **OpenAPI regen then ignore wrapper types** — `pnpm openapi-codegen` can renumber Hey API
  `SchemaFooN` exports; hand wrappers in `libraries/shared/types/chatTypes.ts` /
  `emailTypes.ts` break with TS2724. Grep those files for `Schema` and align to current
  `types.gen.ts` before claiming type-check done (`mock-api-openapi-sync` / `rest-api-workflow`).
