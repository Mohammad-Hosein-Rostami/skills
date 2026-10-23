Baseline skills (load first): skill-first-governance, fullstack-feature-delivery-loop, figma-feature-delivery

ICE API — skills: rest-api-workflow, api-contract-first, backend-api-reliability, toast-copy-standards

Scope fence: this prompt owns `https://ice-api.desap.net` only. OpenAPI input is `https://ice-api.desap.net/api/v1/openapi.json`. Generate with `pnpm openapi-codegen:ice` into `libraries/shared/services/ice-api-client` (`openapi-ts.ice.config.ts`). Never hand-edit that output. Never run the mock multipart patch script on it. Never retarget `pnpm openapi-codegen`, `openapi-ts.config.ts`, or `libraries/shared/services/hey-api-client`.

`VITE_API_URL` is the ICE API base. `VITE_MOCK_API_URL` and `useApi` / `callApi` stay on the mock for features that have not been migrated. Do not switch an unmigrated screen to the ICE API client. Do not put the Ava Olivia one-click on the public login button; that path stays dev-only (`02-api-mock.md` / `03-api-mock-delivery.md`).

- confirm the endpoint exists on the ICE API OpenAPI before coding a call
- call it through the ICE API client with `VITE_API_URL` (credentials include when the contract uses the ice_session cookie)
- use generated ICE API types; do not cast mock types onto ICE API payloads
- map `error.key` to i18n; the ICE API envelope has no user-facing message string
- start migrated work at login (`POST /api/v1/auth/login`, MFA channel, MFA OTP) and leave the rest of the portal on the mock client until that section is explicitly migrated
- if the ICE API fails, report the status and error key; do not “fix” it by falling back to the mock login or by editing the mock client
- regenerate ICE API types when the deployed OpenAPI changes (`pnpm openapi-codegen:ice`, which also strips the stale `@ts-expect-error` in the fetch client), then re-check call sites that import `ice-api-client`
