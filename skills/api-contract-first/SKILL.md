---
name: api-contract-first
description: Treat OpenAPI /ice-calls and useApi endpoints as the production contract even when the backend is not deployed yet. Use for ICE Calls REST work, parity audits, and any feature where agents must not dismiss APIs as temporary mocks.
---

# API Contract First

**Use when** working on `pages/iceCalls/**` or any feature whose REST surface is defined in OpenAPI / Hey API but the server is not live yet.

Pair with **`rest-api-workflow`** and **`ice-calls-jitsi`** for ICE Calls.

## Core principle

**The API contract is real.** Endpoints, paths, request bodies, response shapes, and `useApi` wiring are the **intended production integration**. The backend may be unimplemented today; the frontend must still:

- Call the correct endpoint via `useApi` / `callApi`
- Use generated types from `hey-api-client`
- Handle loading, success, empty, and error states as production UI
- Refresh lists/details after mutations the same way production will

## Language agents must use

| Say                                         | Do not say (for REST/API) |
| ------------------------------------------- | ------------------------- |
| production API contract                     | mock API                  |
| OpenAPI contract                            | fake endpoint             |
| backend not yet deployed                    | mock backend              |
| contract-aligned UI                         | placeholder API           |
| dev user identity shim (`iceCallsMockAuth`) | mock auth for APIs        |

**Exception — dev-only UI preview (not API):** parity capture seeds and local in-call preview helpers (files like `iceCallsCallCoreMock.ts`, `iceCallsInCallMock.ts`) may be described as **dev UI preview**, **parity capture**, or **local preview state**. They simulate **UI state for Figma verification**, not substitute API contracts.

## ICE Calls `/ice-calls/*`

- Index: `docs/ice-calls/README.md`
- All integration via `useApi` from `@ice-web-app/shared-hooks`
- Do **not** replace contract calls with hardcoded JSON blobs in components
- Do **not** ship `DEMO_*` / `FAKE_*` / fixture event or holiday arrays in calendar UI — empty API → empty UI
- Do **not** leave calendar hooks as stubs (`useIceCallsCalendar` must call `GET /ice-calls/calendar`)
- Do **not** fall back to calendar/day/week/month **fake fixtures** when API data is empty — render empty boards
- After schedule mutations, always invalidate **events + calendar + calls** together via **`evictIceCallsEventSyncCache`** (prefix) — never exact-key `evictApiCache` with a single pagination query
- Prefer an explicit board/drawer `refresh()` / `onSuccess` / `onMutated` in addition to eviction when those surfaces are mounted
- Recurrence: trust **`GET /ice-calls/calendar` expansion** of `repeatType` — do not invent client-only occurrence grids; strip occurrence board ids with `calendarMasterEventId` before `update-event`
- Never allow create/reschedule into the past (API + FE)
- Day/week boards use **`buildCalendarBoardHours(8)`** → first row **8 am**, overnight **12 am–7 am** at the bottom — do not open with 12 am at the top and scroll to 8 am
- Do **not** skip error handling because “backend isn’t ready”
- Do **not** add GraphQL — it has been removed from the repo
- Do **not** add zone-specific request behavior to global shared API internals (`useApi.utils`, global fetch wrappers) without explicit user approval.
- Do **not** create dual production/mock API code paths by default. The one exception is the ICE API client beside the mock client (`rest-api-workflow` — ICE API vs mock). Do not collapse that split, and do not name the ICE API side `real`.

## Backend-first improvement handoff (ICE Calls)

Before implementing major ICE Calls slices, produce or update:


Each item must include:

- endpoint/contract surface
- current issue
- proposed backend change
- frontend impact
- priority

## When backend returns errors

- Show user-facing error via area toast pattern (`showCallToast`, `toast.error`)
- Response `message` on success responses mirrors FE i18n English (`toast-copy-standards`) — mobile/notifications may display API text directly
- Document blocker in report — **do not** rip out `useApi` and inline data
- Empty states use real response shape (`data: []`, `total_count: 0`)
- Improve backend responses for known failures (typed 4xx/409, clear detail) instead of generic `500` when you are editing backend code.
- Keep UX copy specific and actionable; avoid raw server text unless explicitly mapped.

## Async state UX expectation

Contract-first includes transition states, not only request/response wiring.

- For long async chains, provide progress/processing UI and status text.
- Keep action controls disabled/replaced until terminal state.
- Use blocking loading only for first-load/recovery; prefer silent refresh for realtime sync.

## Audit checklist

| Check                 | Pass criteria                                    |
| --------------------- | ------------------------------------------------ |
| Endpoint path         | Matches OpenAPI / docs index                     |
| Method                | GET vs POST/PUT correct                          |
| `enabled` / `execute` | Matches area conventions                         |
| Types                 | From generated client, not `any`                 |
| Mutation refresh      | `useCallApiData.execute`, hub list refresh, etc. |
| i18n errors           | No raw server messages in UI without mapping     |
| No GraphQL            | GraphQL/Apollo removed from the repo             |

## Mistakes

- “Backend isn’t ready so I’ll hardcode the list”
- Calling REST “mock” in docs, comments, PR text, or prompts
- Removing `useApi` because fetch fails in dev
- Hand-editing `hey-api-client/**`
- Using temporary FE workarounds as excuses to bypass contract wiring
- Introducing shared/global API changes for an ICE-calls-only problem
