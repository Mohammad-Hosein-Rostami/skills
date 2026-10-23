Baseline skills (load first): skill-first-governance, fullstack-feature-delivery-loop, figma-feature-delivery

API contract & mock sync — skills: mock-api-openapi-sync, rest-api-workflow, api-contract-first, backend-api-reliability

Scope fence: this prompt owns the mock API only (`ice-mock-api`, `VITE_MOCK_API_URL`, generated client `libraries/shared/services/hey-api-client`). Do not change `VITE_API_URL`, `openapi-ts.ice.config.ts`, or `libraries/shared/services/ice-api-client`. Do not point `useApi` / `callApi` at the ICE API. Public login stays on the ICE API client (`08-api-ice.md`).

- created api must be synced with other related parts of the app /.../ice-mock-api
- datas must be fake but should be synced with other sections
- you must use openapi auto generate after completing api in frontend side (`pnpm openapi-codegen` only — never `openapi-codegen:ice`)
- check the needs of front end for what apis it needs
- then start creating needed apis
- sync apis with other backend sections
- always update schema.md file of that section after creating an api
- always update schema.md file of that section if you update any api
- use local auto generate types
- after generating types revert local urls
- consider good short and meaningful response messages
