---
name: nx-ice-verify
description: ICE monorepo verify loop for agent PRs — nx affected lint/type-check/build, ESLint/Prettier vs boundaries, project type-check names. Use after edits, before claiming done, or when CI/local verify fails.
---

# ICE Nx Verify

## Default agent verify (honest PR loop)

```bash
pnpm verify:affected
```

Runs `nx affected` for `lint`, `type-check`, and `build` (see root `package.json`).

Also run architecture boundaries when imports/layers changed:

```bash
pnpm boundaries
```

## Lint split (do not confuse)

| Command                          | Tool                | Purpose                                         |
| -------------------------------- | ------------------- | ----------------------------------------------- |
| `pnpm lint` / `pnpm lint:staged` | **ESLint**          | Lint (source of truth with Prettier for format) |
| `pnpm boundaries`                | **Nx graph** + tags | Layer/tag constraints                           |
| `pnpm e2e:agent:smoke`           | **Playwright**      | Optional UI smoke (`agent-browser`)             |

Pre-commit uses lint-staged (Prettier + ESLint). Boundaries should pass before merge when touching cross-project imports.

For interactive UI verification use skill **`playwright-agent-eyes`** (Playwright MCP) — not part of `verify:affected`.

## Focused checks

```bash
pnpm nx show projects --withTarget type-check
pnpm nx run <project>:type-check
pnpm nx run <project>:lint
pnpm nx run <project>:build
```

After feature edits, **also** gate the touch set (see `ai/prompts/06-type-check.md` /
`eslint-and-verification`):

```bash
pnpm exec eslint --max-warnings=0 <touched-paths>
# + scoped tsc diagnostics / ReadLints on each touched .ts/.tsx
```

Do not mark type-check done solely because a failing project `tsc` log has no lines for your paths.

### Project names (Nx)

- Host: `portal`
- Ecosystems: `innovation`, `nomad`, `school`
- Zones: `collaborationZone`, `challengeZone`, `contentZone`, `defaultSchoolZone`, `timeTableZone`, `workflowZone`
- Shared Nx: `designSystem`, `ui`
- E2E: `agent-browser`

Many `libraries/shared/*` folders are **aliases only** (not Nx projects) — verify via consuming project's type-check + root ESLint/Prettier.

## Explore first if unsure

Load **`nx-workspace`** / **`nx-run-tasks`** skills. Use:

```bash
pnpm nx show project <name>
pnpm nx show projects -p 'tag:type:zone'
pnpm nx graph --file=/tmp/nx-graph.html
```

## Never

- Claim “verified” after only reading code
- Run globally installed `nx` / `npm` for this repo
- Skip `boundaries` after adding cross-layer imports
- Add Storybook or GraphQL as part of verify
