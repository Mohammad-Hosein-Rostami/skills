---
name: ice-calls-jitsi
description: Primary workflow for ICE Calls and custom lib-jitsi-meet. Use for pages/iceCalls, callSession/media/ui stores, Jitsi connect/join/leave, in-call UI, and /ice-calls REST in collaborationZone.
---

# ICE Calls / Jitsi

Primary skill for `libraries/zones/collaborationZone/src/pages/iceCalls/**`.

References:

- `docs/ice-calls/README.md` — calling-system documentation (chapters 01–13; layouts = `06-layouts-and-reactions.md`, troubleshooting = `10-troubleshooting.md`). Behaviour changes update the matching chapter in the same change; check with `pnpm docs:ice-calls:verify`.
- `ice-calls-completeness-plans/000-START-HERE-constitution.md`

Pair with `rest-api-workflow`, `api-contract-first`, and `backend-api-reliability` when backend endpoints are involved.

## Core principles

1. **Production-path first**: no separate mock-mode architecture unless user explicitly requests temporary preview mode.
2. **Zone-local runtime**: Jitsi runtime stays inside `pages/iceCalls/**` (`runtime/`, stores, call UI).
3. **Shared safety**: avoid changing global/shared API/design-system infrastructure for ICE Calls without user approval.
4. **Fetch discipline**: one-shot fetch + explicit refresh; only poll where lifecycle requires it.
5. **Prime Directive**: do not change UI pixels or user flows unless product explicitly asks.

## Hard rules (rebuild)

1. **No Jitsi objects in reactive state for rendering** — `JitsiConnection` / `JitsiConference` / tracks live in `callRuntime`; stores hold view-models and flags.
2. **No listener outside the event bridge** — register via `SubscriptionScope` in `runtime/eventBridge/**` (or another scoped runtime owner).
3. **No `srcObject` outside the PiP adapter** in call UI — tiles use `MediaSink` / `bindTrackToElement`.
4. **No store hook without a selector** — never bare `useCallSessionStore()` / `useCallMediaStore()` / `useCallUiStore()`.
5. **Never use display name as identity** — resolve endpoint → ICE user via the identity map (Plan 12 option A).
6. **Log with `createLogger(namespace)`** — never log tokens, JWTs, emails, or full user objects. Override: `localStorage.setItem('ice-calls:log', 'tracks,quality:debug')`.
7. **Dev diagnostics only** — `?iceCallsDiag=1` or Ctrl+Shift+D; must not ship in production bundles.

## Integration contract

- Tarball `lib-jitsi-meet` GitHub release (v2212) is the source; import ESM via `runtime/jitsiMeetJs.ts`.
- UI consumes **named selectors** on `useCallSessionStore` / `useCallMediaStore` / `useCallUiStore`.
- Realtime: one `createRealtimeCoordinator` per call (WS ticket + versioned merge + media-state).

## Lifecycle baseline

1. Hub route loads scheduled/history lists.
2. Start call / go-to-call action sets `callId`, `roomName`, `callType`.
3. **Pre-join** (`SyncPreJoin`): fetch call + JWT, `initJitsi`, `ensureLocalTracks` only — no XMPP connect yet.
4. Join flow: `POST join-call` (Idempotency-Key) → on `joinState: JOINED`, connect + join; on `WAITING_FOR_HOST`, toast and stay on prejoin.
5. In-call UI (`SyncCall` after joined): reconnect/connect for active conference.
6. End/leave through REST then scoped teardown.
7. Floating/minimized state is route-safe and non-invasive.

## Jitsi server config (ice-call.desap.net)

Match `constants/jitsiConfigs.ts` to live `https://ice-call.desap.net/config.js`:

- `hosts.domain`: `meet.jitsi`
- `hosts.muc`: `muc.meet.jitsi`
- `serviceUrl`: `wss://ice-call.desap.net/xmpp-websocket`
- JWT minted by backend: `iss`/`JWT_APP_ID` = `ice-portal`, `aud` = `jitsi`, `sub` = `meet.jitsi`
- Must match Jitsi server `JWT_APP_ID`, `JWT_APP_SECRET`, `JWT_ACCEPTED_ISSUERS`, `JWT_ACCEPTED_AUDIENCES`
- Do not use `JWT_AUDIENCE` on Jitsi docker env; the accepted audience variable is `JWT_ACCEPTED_AUDIENCES`

Backend env on mock API / Dokploy (never commit secret):

- `JITSI_JWT_APP_ID=ice-portal`
- `JITSI_JWT_AUDIENCE=jitsi`
- `JITSI_JWT_APP_SECRET=<same as ice-call.desap.net JWT_APP_SECRET>`

## JWT auth troubleshooting baseline (required for auth failures)

When FE shows `connection.passwordRequired` or `Strophe authfail`, validate server-side JWT wiring before changing frontend logic.

1. Jitsi env updates must be applied with recreate:
   - `docker compose up -d --force-recreate prosody jicofo web`
2. Validate generated Prosody config (source of truth):
   - `/config/conf.d/jitsi-meet.cfg.lua` must show:
     - `authentication = "token"` (on `meet.jitsi`)
     - `app_id = "ice-portal"`
     - `app_secret = "<expected>"`
     - `asap_accepted_issuers = { "ice-portal" }`
     - `asap_accepted_audiences = { "jitsi" }`
3. `printenv` on the host shell is not sufficient; check inside container or generated Prosody config.
4. If error is `Invalid signature`, compare signer/verifier secret fingerprints (length + short hash) between:
   - mock API `JITSI_JWT_APP_SECRET`
   - Prosody `app_secret` in generated `jitsi-meet.cfg.lua`
5. `authentication = "jitsi-anonymous"` and `authentication = "internal_hashed"` are expected for guest/auth virtual hosts; only `meet.jitsi` should be `token`.

## Hard-won rules (post-completeness)

1. **Olm before the connection.** `ensureOlmReady()` must complete before `createJitsiConnection` — LJM only advertises `FEATURE_E2EE` if `window.Olm` exists at XMPP construction. Never move it back into join.
2. **GET refetches need `evictApiCache`.** `callApi` caches GETs for 5 min; identities (and any "must be fresh" read) evict first. The server's `call.participant.identity` WS push is best-effort — poll until bound. Eviction lands on the next microtask (plan 3108 makes it synchronous).
3. **Tiles read mute from the store** (`selectIsTrackMuted`), not only from the runtime track.
4. **Meeting-settings is PUT-only.**
5. **A call never changes type.** `callType` decides the view; a ONE_TO_ONE with added people stays ONE_TO_ONE (same name, contact info, classic view) — they are visitors (`Call.primaryPeerIds` = the original pair), shown in the participants list, the 1:1 side column and the Grid option. Never reintroduce a 1:1→GROUP flip, a sustain timer, or group chrome for a 1:1.
6. **Debug media/avatars with data, not layout edits:** check per-client E2EE live state, `window.__ICE_CALLS_TRACKS__()`, and the identity map before touching layout/`srcObject` code. Never clear `srcObject` or unbind hidden layouts as a fix (it blanked live tiles).
7. **Verify with the multi-user harness** (`@third-join-media`) after any change to media, identity, layout or E2EE code.
8. **Group layout = one engine.** `runtime/layout/computeGroupLayout.ts` is pure (no React/store/Jitsi); vocabulary `one-to-one|grid|speaker|sidebar|duo|spotlight|pinned|share` (`data-layout`). Pins: latest pin wins the stage; 1 pin in Speaker/Sidebar/Spotlight, ≤ 2 elsewhere. Auto picks `duo` for ≥ 2 recent remote speakers (10 s). Tile overlays (speaking/focus ring) go on `::after` — positioned card content paints over an inset outline.

9. **The quality controller must be running.** `startQualityController(scope)` (receiver/sender constraints, lastN, user quality prefs, the degrade ladder) is started in `store/getRoomSlice.ts` next to `attachConferenceEvents`. It silently never ran after the store refactor because its old starter had no caller — after touching session wiring, assert `getQualityControllerDebug().hasController` in a joined call.
10. **Server truth decides outcomes; the FE renders.** History result/tone/reason (`historyResult*`), caller outcomes (`endReason`), lobby queue, recording state and role come from the backend. Never derive "declined / missed / busy / recording" from presence or local guesses. Badge on an avatar = call status **or** presence, never both.
11. **Encryption is never over-claimed.** The green E2EE chip shows only when live **and** every remote peer supports it (`resolveE2eeNotice`); a browser without E2EE is told before joining, not blocked and not silently downgraded. Toggling E2EE in place after sessions exist breaks Olm — heal a dark remote video with a rejoin (`requestReconnect('manual')`), not a key toggle.
12. **A GROUP room stays open when its only occupant leaves; a 1:1 ringing call is cancelled.** A 1:1 to a busy callee is refused up front (`409 call.peer_busy`) — there is no "rejected" history row to assert on. Group meeting settings default `lobbyEnabled: true`: specs that need a second member to join directly `PUT {lobbyEnabled:false}`.
13. **Everyone must see that a call is recorded.** `RecordingActiveOverlay` (stage, top-right) and the header timer are not optional chrome; recordings are accessible to hosts, the caller and attendees only.
14. **Realtime from worker threads goes through `loop_bridge.py`.** The reaper/scheduler have no event loop; hubs are process singletons (`sys.modules`). Never create a second hub instance.
15. **Specs: switch users with `switchDevUser` (it re-pins the context), pass the API base from `ICE_CALLS_MOCK_API_BASE`, never sniff it from resource timings, never edit portal source during a live run (HMR splash), and treat "did not run" as failure (`reporters/strict-skips.ts`).** Restart the portal with `VITE_API_URL`/`VITE_MOCK_API_URL` when the API under test is local; stale module copies can hide controller state.
16. **Docs move with behaviour.** Run `pnpm docs:ice-calls:verify`, `pnpm i18n:ice-calls:check` (en/ar/de parity) and `pnpm ice-calls:raw-fetch` (no new raw `fetch` — the generated client is regenerated from a deployed spec, never hand-edited) before handing over.

## Guardrails from retrospective

- Do not add ICE-Calls-specific wrappers in `libraries/shared/hooks/useApi.utils.ts`.
- Do not add global route/component changes (for example always-mounted floating call) without explicit approval.
- Do not introduce `*Mock*.ts` data paths/query flags unless explicitly requested by user.
- Keep toast behavior minimal and consistent with existing containers/patterns.
- **Recording feedback is two Figma variants — never one chrome:**
  - Started (`22542:413614`): Banner, `success.opacity-16`, outline `CircleCheck`, green copy,
    privacy link + close → `variant: 'started'` / `ice-calls-recording-started-toast`
  - Saved (`16228:392803`): standard toast, `success.opacity-8` + md shadow, `CircleCheckFill`,
    iceGray copy, no close → `variant: 'saved'` / `ice-calls-recording-saved-toast`
  - See `RecordingCallToast.tsx` + `toast-custom.css`. Do not re-merge classes or icons.
- For unusual/ambiguous Figma patterns, ask user before implementing structural UI changes.

## API usage rules

- Always use `useApi` / `callApi` for `/ice-calls/*`.
- Treat OpenAPI as real production contract.
- Mutating join/end/media-state: send `Idempotency-Key` created once per user action.
- Map `ApiError.code` via `runtime/api/iceCallsErrorMap` — toast **copy** stays i18n as today.
- Refresh intentionally after mutations; prefer silent refetch for WS-driven updates.
