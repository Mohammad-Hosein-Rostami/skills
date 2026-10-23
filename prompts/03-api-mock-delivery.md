Baseline skills (load first): skill-first-governance, fullstack-feature-delivery-loop, figma-feature-delivery

Mock API delivery loop — skills: mock-api-openapi-sync, rest-api-workflow, api-contract-first, playwright-agent-eyes, playwright-verify-fix-loop

Scope fence: “backend” here means ice-mock-api only. Types stay in `libraries/shared/services/hey-api-client`. Runtime for these features stays on `VITE_MOCK_API_URL`. Do not edit `VITE_API_URL`, `openapi-ts.ice.config.ts`, or `ice-api-client`. Do not replace the public `/login` flow with mock one-click auth. ICE API work uses `08-api-ice.md`.

- analyze the target section
- collect where that section needs an api
- design the needed apis and keep them synced with related sections
- auto generate api types locally (`pnpm openapi-codegen` against ice-mock-api)
- implement the apis on the mock API
- use relevant toasts for api responses
- write tests for all api scenarios
- test the apis with playwright in an opened browser so i can see it
- use MCP for headed verification
- if there is an issue, fix it in the mock backend
- auto generate types locally again
- apply the changes on the section
- repeat until the flow works end to end
