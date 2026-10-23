---
name: rest-api-workflow
description: Primary API skill for this repo. Use for all new data access — REST calls, useApi, callApi, Hey API client, OpenAPI codegen, upload/download flows.
---

# REST API Workflow

**Default API skill.** REST/OpenAPI is the only data-access path.

See also: `docs/graphql-to-rest-migration.md` (GraphQL removed).

## Stack (in order)

1. `useApi` — feature hooks/components
2. `callApi` — imperative calls (supports `skipJsonContentType` for multipart)
3. Generated client: `libraries/shared/services/hey-api-client`
4. Shared helpers: `uploadGeneralFiles`, `getImageApi` (`@ice-web-app/shared-helpers`)
5. Auth refresh only: `libraries/shared/services/refreshApiToken.ts` (used by `useApi`/`callApi`, not a general client)

**Removed:** `apiService` / `libraries/shared/services/api.ts` / `fetchFileNewTab.ts`. Do not reintroduce them.

## Workflow

1. Confirm endpoint exists in Hey API generated surface
2. Reuse shared auth/header/context handling — no raw `fetch` in feature code
3. Uploads to `/api/general/upload` → `uploadGeneralFiles` (metadata JSON + unwrap `.data`)
4. Authenticated image/file blobs → `getImageApi` (`GET /downloads/{relative_path}` on the callApi / `VITE_MOCK_API_URL` host). Stored HTML may contain `https://<VITE_API_URL>/api/general/downloads/{relative}` from `resolveUploadPathToUrl` — that host often 200s with an empty body. Strip to `{relative}` (skip `avatar/`, which stays on `GET /api/general/downloads/avatar/{file_name}`) and fetch `/downloads/{relative}`. Do not `fetch()` a non-mock absolute downloads URL; portal API keys must stay on `https://ice-mock-api.desap.net`.
   - **`DsAvatar` / `DsSelect` person rows:** pass `imgUrl: profilePicture` (and `alt`), never `src: profilePicture`. Relative `downloads/profile-picture/…` 404s on the portal origin and falls back to initials. `DsSelect` `Tag` must spread `option.avatar` so chips also get `imgUrl`. Email compose (`ComposeRecipientField`) is the existing pattern.
5. If OpenAPI changed: regenerate, don't hand-edit feature workarounds
6. After regen: update any `libraries/shared/types/*` re-exports that pin `SchemaFooN` names from
   `types.gen.ts` (Hey API renumbers colliding schemas). Fix the import — do not `as` cast.

```bash
# Prefer local mock when contract changed but not yet deployed:
# openapi-ts.config.ts input → https://ice-mock-api.desap.net/openapi.json
pnpm openapi-codegen
# Then: rg "Schema[A-Za-z]+[0-9]+" libraries/shared/types — confirm each still exports from types.gen.ts
```

`openapi-codegen` runs `@hey-api/openapi-ts` then `openapi-codegen:types`. The types step:

- accepts single- or double-quoted `url:` strings in `types.gen.ts`
- bounds each `*Data` type to the next `export type`/`export const` so models that only _end in_ `Data` (e.g. `CardResponseWithData`) cannot steal the next endpoint `url`
- keeps generated client `baseUrl` on `https://ice-mock-api.desap.net` in `client.gen.ts` / `types.gen.ts`
- patches known multipart upload fields to `Blob | File` (required) or `Blob | File | null` (nullable). Deployed OpenAPI sometimes types these as `string` / `Array<string>`.

Add new multipart fields to that patch list in `scripts/generate-api-types.ts` when needed — do not `as File` in feature code.

Never call generated SDK functions from feature code without `createApiClient()` (or, preferably, `useApi` / `callApi`). The SDK default client is not the env base URL.

## ICE API vs mock

Two clients. Do not merge them, and do not label the ICE API side `real`.

|         | ICE API (migrated calls, including public `/login`)                                         | Mock (everything not migrated)                                |
| ------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Env     | `VITE_API_URL` = `https://ice-api.desap.net`                                                | `VITE_MOCK_API_URL` = `https://ice-mock-api.desap.net`        |
| Client  | `libraries/shared/services/ice-api-client` via `getIceApiClient()`                          | `hey-api-client` via `useApi` / `callApi` / `createApiClient` |
| Codegen | `pnpm openapi-codegen:ice` (`openapi-ts.ice.config.ts` + `scripts/patch-ice-api-client.ts`) | `pnpm openapi-codegen`                                        |

Naming: the default path has no `real` prefix (`useLogin`, `startLogin`, `ice-api-client`, `getIceApiClient`). Only the mock duplicate is marked `mock` (`DevMockLoginButton`, `VITE_MOCK_API_URL`). Do not rename the existing mock `hey-api-client` folder or the `openapi-codegen` script.

- Never `export *` from `ice-api-client` (it collides with `hey-api-client`, especially `client`).
- Never run `scripts/generate-api-types.ts` on `ice-api-client`. `patch-ice-api-client.ts` only strips a stale `@ts-expect-error` above `beforeRequest`.
- `useApi` reads only `VITE_MOCK_API_URL`. Public `/login` uses `getIceApiClient` and must not call `getBackendBaseUrl` or set mock `isAuthenticated`.
- Never point codegen or `VITE_API_URL` at ice-api-dev (CORS blocks localhost). Set `VITE_API_URL` in both `.env` and `.env.development`. A running `nx serve` keeps the old inlined value until a full process restart — Vite dotenv does not override an existing `process.env`.
- Login result unions: `if (!result.ok)` does not narrow under portal `tsc`. Use `isLoginFailure`.
- Do not use `TelephoneInput` (`type="tel"`) on public login — it calls mock registration. Do not use `useLanguage` there — it persists through the authenticated API.

## Errors

Match surrounding pattern — usually `try/catch` + `toast.error(...)`.

For **ICE Calls** (`/ice-calls/*`):

- Prefer the uniform `ApiError` envelope `{ code, message, details, retryable, requestId }`.
- Map `code` → i18n via zone `iceCallsErrorMap` (do not change toast copy).
- Log `requestId` on failures for support traces.
- User-initiated mutations that may retry (join/end/media-state) must send an `Idempotency-Key`
  created **once per user action** and reused across retries of that same action.

## Mutation toasts

Load **`toast-copy-standards`** when wiring success/error feedback after `useApi` / `callApi` mutations:

- Success: `toast.success(t(...))` — i18n-first; do not prefer `response.message` for routine actions
- Errors: contextual key or `apiErrorMessage(error) || t('ACTION_FAILED')`
- Mock API `message=` must match the same English as the i18n key; update `schema.md` together

Quality bar:

- Prefer specific, user-safe messages over generic failures.
- Map common error classes (auth/permission/validation/conflict/timeout/server) to distinct UX copy.
- Do not leave known failures as opaque `500` responses; when editing backend, return typed errors where possible.

## Async UX

For long-running actions (join/connect/submit/upload):

- Disable or replace repeatable action controls while in-flight.
- Show contextual progress text ("what is happening now").
- Keep progress visible until a terminal state is reached.
- Avoid re-showing primary action controls before the async chain has actually completed.

## Realtime Refresh and Loading

- Separate initial load from background sync.
- Background websocket/invalidation refresh should prefer silent refetch and keep existing UI visible.
- Reserve blocking overlays for first-load and broken/recovery states.

## List refresh after mutation / OAuth popup

`useApi` GET responses are cached. After a mutation (or OAuth popup `postMessage` success) that should
change a list:

1. `await invalidateApiQueries([{ endpointPrefix, method: 'GET' }], { flushSync: true })` (or the
   shared helper that wraps it)
2. Then `execute()` / refetch — **refetch alone often returns the previous cache**

Account Connections (`/account/connection`) is the reference: OAuth stays in a popup so the parent
keeps mounted; invalidate email-accounts GET before refetching My Email Accounts.

## Mistakes

- Reintroducing GraphQL/Apollo instead of REST
- Raw `fetch` when `useApi` / `callApi` / helpers fit
- Reintroducing `apiService` or a parallel HTTP client
- Hand-editing `hey-api-client/**` output (use regen + `generate-api-types.ts` patches for binary uploads)
- Duplicating headers auth helpers already provide
- Triggering loading overlays on every realtime refresh
- Debugging refetch storms without enumerating all trigger sources first
- Calling generated SDK functions without `createApiClient()` / `useApi` (hits leftover codegen `baseUrl`, often `ice-mock-api.desap.net`)
- Prefixing the ICE API client, hooks, or codegen with `real` (`useRealLogin`, `hey-api-client-real`, `openapi-codegen:real`)
- Pointing `useApi` / `pnpm openapi-codegen` at the ICE API, or falling back to mock login when the ICE API fails
- Uploading with flat FormData `type`/`threadId` fields instead of mock `metadata` JSON
- Putting `src: contact.profilePicture` on `DsAvatar` / `DsSelect` options — use `imgUrl` so `getImageApi` authenticates `downloads/…`
- Calling `refetch`/`execute` after OAuth or mutation without `invalidateApiQueries` (+ often
  `flushSync`) — list UI stays stale until a full remount
- Leaving `SchemaUserInfo2` / `SchemaThreadsResponse2` (etc.) in `libraries/shared/types/*` after
  OpenAPI regen renumbered them — fix imports to the current `types.gen` export
- Full-page `window.location` for provider OAuth when the opener must refresh a `useApi` list in-place
  (prefer popup + `postMessage` / `sessionStorage`)
