---
name: graphql-workflow
description: RETIRED — GraphQL/Apollo has been removed from this repo. Use rest-api-workflow for all API work.
---

# GraphQL Workflow (Retired)

**GraphQL has been removed** from this monorepo (Apollo packages, codegen, `graphQueries`, providers, `VITE_GRAPHQL_*`).

Do **not** follow any legacy GraphQL maintenance steps below as if GraphQL still exists.

## What To Do Instead

1. Load and follow **`rest-api-workflow`**.
2. Use `useApi` / `callApi` / Hey API client (`pnpm openapi-codegen`).
3. For screens that still show empty/fake data after GraphQL removal, wire them to REST — see root `graphql-usage-list.txt`.
4. Collaboration contacts/groups folders are `iceContacts` and `collabGroup` (not `*V2` / V1 GraphQL trees).

## Historical note

`docs/graphql-to-rest-migration.md` is retained as a short “migration complete” notice only.
