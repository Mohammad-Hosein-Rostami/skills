---
name: mock-api-openapi-sync
description: Design and implement ice-mock-api endpoints synced with related sections, update schema.md, local OpenAPI codegen, revert local URLs, and wire frontend via useApi. Use when adding/changing mock APIs for a feature alongside ice-web-app.
---

# Mock API ↔ OpenAPI ↔ Frontend Sync

Companion: **`api-contract-first`** · **`rest-api-workflow`** · **`backend-api-reliability`** · **`toast-copy-standards`**.  
Rules: `generated-files-boundaries.mdc` · `openapi-nullability-boundaries.mdc` · `async-ux-and-debug-playbook.mdc`.

| Role     | Path                                                                               |
| -------- | ---------------------------------------------------------------------------------- |
| Mock API | `/Users/mohammad-hosein-rostami/Desktop/Desap/ice-mock-api` (or workspace sibling) |
| Frontend | ice-web-app — Hey API client under `libraries/shared/services/hey-api-client`      |

## Order (do not invert)

1. **Inventory frontend needs** from the real UI/flow (loading, empty, success, validation, permission, conflict).
2. **Design contracts** aligned with related mock-api domains (same entity IDs, relations, field names).
3. **Implement** in ice-mock-api with short, meaningful, user-safe response messages.
4. **Sync fake data** across related sections — consistent IDs/names; no contradictory fixtures.
5. **Update that section’s `schema.md`** on every create or change.
6. **OpenAPI codegen** in frontend (`pnpm openapi-codegen` — input stays on `https://ice-mock-api.desap.net/openapi.json`).
7. **Keep committed env/codegen** on the deployed mock URL — never a machine-local mock base.
8. **Wire FE** with `useApi` / `callApi` + generated types — no raw `fetch`, no hand-edit `hey-api-client/**`.
9. **After codegen:** fix hand-written wrappers under `libraries/shared/types/*` that import Hey API
   `SchemaFooN` aliases — OpenAPI regen often **renumbers** them (`SchemaUserInfo2` → `SchemaUserInfo1`).
   Grep for the removed export; update imports to the current `types.gen.ts` name — never `as`.
10. **Toasts** — i18n-first on FE; API `message` mirrors English i18n (`toast-copy-standards`).
11. **Headed verify** (`playwright-agent-eyes` → `playwright-verify-fix-loop`); fix backend root cause when contract/data is wrong; regen; repeat.

## Quality bar

- Treat OpenAPI as the production contract (`api-contract-first`) even when data is fake.
- Explicit 4xx for business failures; reserve 5xx for unexpected faults (`backend-api-reliability`).
- Prefer specific toast copy over generic “Something went wrong”.
- Nullability: FastAPI `Optional` → `T | null`; widen UI props — never paper with `as`.

## Do not

- Ship FE against guessed shapes without OpenAPI regen
- Leave `openapi-ts.config.ts` / `.env` `VITE_MOCK_API_URL` on any machine-local mock URL in committed trees
- Edit `events.json` / other `app/data/*.json` and expect live handlers to see it without **restarting uvicorn** — `json_cache` loads at startup
- Map calendar `notes` from unrelated fields (e.g. meeting `password`) in projections — use the real `notes` column
- Reuse schedule-meeting drawers for Figma task chrome — calendar tasks use `CreateAndEditTaskDrawer`
- Ship task `TaskStatus` without Figma’s full set — status dropdown `578:94923` needs `NEW | IN_PROGRESS | DONE | OVERDUE | ON_HOLD` (label Over Due)
- Reuse schedule-meeting drawers for Figma reminder chrome — calendar reminders use `CreateAndEditReminderDrawer` (Figma `578:93892`)
- Leave `openapi-ts.config` / env pointing at a machine-local mock after codegen
- Hand-edit generated `hey-api-client/**`
- Edit `VITE_API_URL`, `openapi-ts.ice.config.ts`, `ice-api-client`, or `pnpm openapi-codegen:ice`. Those belong to the ICE API client (`rest-api-workflow`). Mock codegen stays `pnpm openapi-codegen` into `hey-api-client`.
- Create orphan endpoints that contradict related list/detail/share fixtures
- Skip `schema.md` updates
- Assume Vite portal is on local mock because `.env.development` says so — **verify** the inlined URL in transformed `getEnvVariable.ts` (`VITE_MOCK_API_URL`). This portal has used `.env` over `.env.development` for `import.meta.env`; if create toasts show stale copy (e.g. old `"Event created successfully."` instead of `ICE_CALLS.NEW_REMINDER_CREATED_SUCCESSFULLY` / `"New Reminder Created"`), the browser is on **remote** mock. Flip `.env` (or align both files), clear `node_modules/.vite/apps/portal`, restart `portal:serve`.
- Treat missing calendar dropdown photos as a mock-file gap before checking FE — `contacts.json` already has `downloads/profile-picture/{userId}/…png` under `uploads/`. Initials mean `DsSelect` passed `src` instead of `imgUrl` / `getImageApi`.
- Hard-delete Account Settings users from `user.json` / `user_me` on `POST /user-me/account-setting/delete` — mock is **audit-only** (`account_deletions.json`); the same seed must remain re-loginable and re-deletable for local demos (no 409 “already deleted” gate).
- Invent FE-only email “sync” (sample addresses, screenshot OAuth chrome, Google JS SDK) for Account
  Connections — provider mailboxes are **backend OAuth** (`…/email-accounts/oauth/start` + callback
  via httpx). When `GOOGLE_OAUTH_*` / `MICROSOFT_OAUTH_*` are empty, do **not** 503 “not configured”:
  auto links one fixed test mailbox per box (`test.connection@gmail.com` / `@icloud.com` /
  `@outlook.com` / `@yahoo.com`) and manual links the submitted address, then returns a same-origin
  `authorizeUrl` with `oauth=success`. Real credentials still use the IdP. Prefer popup return over
  full-page redirect so the list page stays mounted. Do not project Connections into `GET /email/my-emails`.
- Leave `libraries/shared/types/chatTypes.ts` / `emailTypes.ts` (and peers) on stale `Schema*N`
  imports after regen — portal `type-check` fails with TS2724 “Did you mean Schema*N±1?”

## Deliverables

- Endpoints + `schema.md` diff
- Codegen done + local URLs reverted
- FE wiring + toast map
- Headed Playwright evidence for API scenarios

## Seed edits vs running mock

`ice-mock-api` keeps fixtures in an in-memory `json_cache`. Editing `app/data/*.json` on disk does **not** update a running process — **restart uvicorn** (or call whatever reload the cache supports) after seed changes, or API responses stay stale.

Named Figma fixtures (e.g. James Baker, David Knowles) often sort **after** generic `peers[:20]` holder slices — **force-include** those user IDs in holder relations and share **every** toCard for multi-view Switch → `GET …/details/{toUserId}`. Portal must hit the deployed mock (`VITE_MOCK_API_URL=https://ice-mock-api.desap.net`) — do not point at local `:8000`.

### HTTP error envelope (pytest / FE)

`app/main.py` maps `HTTPException` to **`{"message": "<detail string>"}`** for non–ice-calls routes — **not** FastAPI’s default `{"detail": …}`. Pytest and Playwright must assert `body["message"]` (optionally fall back to `detail`). Example: `GET /user-me/memberships` unknown user → `404` + `message: "User profile not found"`.

New account endpoints (e.g. `/user-me/memberships`) need the **deployed** mock once shipped — keep `VITE_MOCK_API_URL` on `https://ice-mock-api.desap.net` for headed verify.
