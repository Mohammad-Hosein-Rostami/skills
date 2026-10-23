---
name: backend-api-reliability
description: Improve backend API reliability and UX-facing error quality. Use when editing endpoint handlers/services, fixing 500 errors, adding realtime sync events, or refining status/transition APIs.
---

# Backend API Reliability

Use this when touching backend routes/services for business flows (join/start/end/sync actions).

## Goals

1. Prevent avoidable `500` errors.
2. Return predictable, typed failure responses for known cases.
3. Keep frontend transitions stable by emitting consistent state changes.

## Error-handling standard

- Use explicit 4xx for business failures (validation/permission/conflict/not-found).
- Reserve 5xx for unexpected faults only.
- Include actionable, user-safe `detail` messages.
- Avoid changing response shape silently; preserve contract unless coordinated.

## Transition/API quality standard

For multi-step actions (`join`, `connect`, `leave`, `end`):

- Derive status variables before they are used in side effects/logging/broadcast.
- Persist state once per step and broadcast once per semantic transition.
- Keep transition outcomes machine-readable (for example: `joinState`).

## Realtime + refetch discipline

- Prefer event names that map directly to frontend refresh intent (`presence_updated`, `call_updated`).
- Include `callId` and `updatedAt` in realtime payloads.
- Do not create event storms from duplicate mutation broadcasts.

## Debug playbook for 500s

1. Reproduce with one endpoint and one payload.
2. Inspect recent edits around variable ordering, conditional branches, and side effects.
3. Verify no variable is read before assignment.
4. Validate branch-specific response objects for missing fields.
5. Add/adjust test payloads or smoke scripts for the failing branch.

## Coordination with frontend

- When backend changes transition semantics, document frontend impact.
- If frontend currently uses generic errors, provide clear error codes/messages to map into better user feedback.
