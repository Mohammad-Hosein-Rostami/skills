---
name: playwright-agent-eyes
description: Drive Playwright MCP + Chrome DevTools MCP for UI verification; promote stable flows to apps/e2e/agent-browser. Use for agent eyes, browser smoke, login/UI bugs, Figma runtime checks.
---

# Playwright Agent Eyes

## When To Use

- Verifying UI after a code change
- Reproducing a frontend bug in a real browser
- Checking login / portal routes under `/portal`
- Figma runtime / pixel parity against a running page
- Deciding whether to promote an exploration to `@playwright/test`

Load with Figma skills when doing design parity **and** runtime check.

**Full chains:** **`figma-playwright-parity`** (implement + verify) · **`playwright-verify-fix-loop`** (test + fix) · **`figma-implicit-spacing`** (gaps MCP missed).

## Prerequisites

1. `pnpm playwright:install` once per machine
2. Portal running: `pnpm nx serve portal` → `http://localhost:4200/portal`
3. Cursor MCP servers `playwright` + `chrome-devtools` enabled (`.cursor/mcp.json`)
4. **Default viewport: 1530×920**, `deviceScaleFactor: 1` (mobile only when user asks)
5. App test attribute is **`data-test`** (MCP + Playwright `testIdAttribute`)

## No separate browser app

Playwright MCP launches its own **headed Chromium**. You do not need BrowserStack or another browser product. Chrome DevTools MCP is complementary (console/network/Lighthouse).

## Mandatory Loop

1. **Navigate** — `browser_navigate` to full URL (`http://localhost:4200/portal/...`)
2. **Viewport** — confirm **1530×920** (`browser_resize` if needed; MCP config already sets it)
3. **Snapshot** — `browser_snapshot` (accessibility tree + refs; optional `boxes` for layout)
4. **Interact** — click/type using **refs from the latest snapshot only**
5. **Re-snapshot** after every meaningful action
6. **Visual / pixel** — `browser_take_screenshot` (`scale: css`) + `browser_evaluate` / `browser_run_code` for `getBoundingClientRect` / `getComputedStyle` vs Figma numbers — do **not** rely on vision alone. For spacing failures, load **`figma-implicit-spacing`** then **`playwright-verify-fix-loop`**.
7. **Show the user** — save PNG under `temp/agent-browser/` and **Read** the image into chat so they see the same frame
8. **Fix code** at the owning layer → repeat from snapshot
9. **Promote** to `apps/e2e/agent-browser/tests/flows/` only when the flow is stable **and** the user wants a lasting test

### MCP tools missing?

Tell the user: Settings → MCP → enable/refresh `playwright`. Meanwhile run:

```bash
pnpm e2e:agent:eyes-smoke
```

Then Read PNGs from `temp/agent-browser/` into chat.

## Do / Don't

| Do                                          | Don't                                                                        |
| ------------------------------------------- | ---------------------------------------------------------------------------- |
| Prefer roles/text/refs                      | Guess brittle CSS selectors                                                  |
| Default **1530×920** + DSF 1                | Use 1440×900 or Retina DSF for Figma parity                                  |
| Save + show screenshots in chat             | Only describe the UI without attaching PNGs when visual matters              |
| Measure boxes/styles for pixel work         | Eyeball screenshots as the only pixel gate                                   |
| Keep smoke suite tiny                       | Mass-add E2E for every PR                                                    |
| Save auth to `.auth/` (gitignored)          | Commit tokens / storage state                                                |
| Use Chrome DevTools MCP for console/network | Dump full DOM into chat                                                      |
| Align with DS + i18n rules                  | Hardcode user-facing strings in tests when asserting copy — prefer role/name |

## Commands

```bash
pnpm e2e:agent:smoke
pnpm e2e:agent:eyes-smoke          # headed Chromium screenshots → temp/agent-browser/
PLAYWRIGHT_SKIP_WEBSERVER=1 pnpm e2e:agent:smoke
pnpm e2e:agent:ui
AGENT_EYES_HEADED=0 pnpm e2e:agent:eyes-smoke   # headless
```

Docs: `docs/agent-browser.md` · Auth: `docs/portal-auth.md`

## Mistakes

- Screenshot-first workflows for _interaction_ (token-heavy; use snapshot refs to click)
- Skipping screenshots when the user asked to _see_ the result
- Clicking stale refs after DOM updates without re-snapshot
- Wrong viewport (not 1530×920) or DSF ≠ 1 during Figma parity
- Starting MCP against production with real credentials without explicit ask
- Adding Playwright specs by default (repo policy: tests only when requested — except explicit agent-browser tasks)
- **React Flow trees (My Memberships):** nodes are not clickable if a parent sets `pointerEvents: 'none'` — set `pointerEvents: 'all'` on `.react-flow__node` / the custom node root. Assert selection reorders ecos (selected centered) after click.
- **ICE Contacts scan-qr e2e:** reusing the same seed `cardId`s after a prior pass hits mock `409 already connected` and skips the share dialog — pick unused persona variants (`… 2` / `… 3`) or reset `ice_contact_card_relation` before re-run
- **MI Contact Cards:** hub KPI → `/management-info/contact-cards?kind=…` (six kinds). Prefer `a[href*="contact-cards?kind="]` over role name `My Card Holder` (sidebar also matches). Mock: `GET /management-information/contact-cards` + live Contact Cards stats. `cards-you-declined` has no Figma pages yet — clone declined shell. Do not leave `href=""` on clickable MI Contact Cards KPIs. **Do not clone Data Contacts drill-down UI** (CompactStatCards kind switcher) — Contact Cards Figma is header view-toggle + Timeline/Cards/Search filter + 265×228 grid (Stack wrap, not fluid Grid).
- **i18n curly apostrophes:** `common.json` may use `’` (U+2019). Role/name regexes with ASCII `'` fail (e.g. “I don’t need this Platform”). Prefer `getByTestId` / radio `value` (`dont_need_platform`) or a regex that allows both apostrophe forms.
- **Account delete goodbye:** `/account/deleted` is a **public** route (outside `RequireAuth`). Assert after delete that the session is cleared and the page still renders (`account-delete-flow.spec.ts`).
