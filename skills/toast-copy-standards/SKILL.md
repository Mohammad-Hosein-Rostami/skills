---
name: toast-copy-standards
description: Journey-doc toast copy rules for frontend i18n toasts and backend/mock API message fields. Load when adding or changing mutation success/error feedback, response message strings, schema.md examples, or e2e toast assertions.
---

# Toast Copy Standards

Companion: **`skill-first-governance`** · **`mock-api-openapi-sync`** · **`rest-api-workflow`**.  

## When to load

- Any `toast.success` / `toast.error` / `toast.warning` / `toast.info`
- Mock API or production API `message=` / response copy
- OpenAPI `schema.md` success-message examples
- Playwright / agent-browser flows asserting toast text

## Copy rules (journey doc)

1. **Short completed-action tense** — no `has been` / `have been`
2. **No stative `is`** for actions (e.g. prefer `Changes Saved` not `Changes are Saved`)
3. **Notify grammar** — `will be Notified that you Declined…` (Title Case on product nouns where Figma uses it)
4. **Consistent phrasing** for the same action across FE, mock API, and e2e
5. **Contextual keys** — prefer feature-specific keys (`ICE_CALLS.*`, `SETUP.TOAST.*`) over generic `ACTION_SUCCESS_TEXT`
6. **No trailing `Successfully`** on routine toasts — reserved for high-value setup only (allow list below)

## Allow list — keep `Successfully`

| Location            | Key / endpoint                                                      | Value                                                                              |
| ------------------- | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| collaborationZone   | `ICE_CONTACTS.EDIT_CARD_MODAL.CARD_CREATED_SUCCESSFULLY`            | Your Contact Card Set up Successfully                                              |
| innovation          | `MY_HUB.EDIT_CARD_MODAL.CARD_CREATED_SUCCESSFULLY`                  | Your Contact Card Set up Successfully                                              |
| mock API            | `POST` create ICE contact card                                      | Your Contact Card Set up Successfully                                              |
| account Connections | `USER_ACCOUNT_SETTINGS.EMAIL_ACCOUNT_ADDED_SUCCESSFULLY`            | Your New Email Account has been Added Successfully                                 |
| account Connections | `USER_ACCOUNT_SETTINGS.EMAIL_ACCOUNT_ADDED_TO_CARD_SUCCESSFULLY`    | Your New Email Account has been Added to Your Card Successfully                    |
| account Connections | `USER_ACCOUNT_SETTINGS.EMAIL_ACCOUNT_SET_DEFAULT_AND_ADDED_TO_CARD` | Your New Email Account has been Set as Default and Added to Your Card Successfully |
| mock API            | `POST /user-me/account-setting/email-accounts` (+ goals)            | same Figma Connections strings above                                               |

Everywhere else: strip `Successfully` from **displayed** copy (key suffix `_SUCCESSFULLY` may remain).

## Frontend implementation

### Library and container

```tsx
import { toast } from '@ice-web-app/shared-utils';
```

Never import `react-toastify` directly in zone/feature code — the portal mounts the shared toast container (`figma-exact-not-similar`).

### Success — i18n-first (required)

For routine mutations, **always** show the i18n string. Do **not** prefer `response.message` / `result?.message` as the primary success source — API messages can drift and bypass locale overlays.

```tsx
// Good
toast.success(tCol('ICE_CALLS.NEW_REMINDER_CREATED_SUCCESSFULLY'));

// Bad — API-first fallback
toast.success(result?.message?.trim() || tCol('...'));
```

### Errors

Prefer contextual keys, then shared fallbacks:

```tsx
toast.error(apiErrorMessage(error) || t('ACTION_FAILED'));
```

Use raw `error.message` only when no contextual key exists and the API returns a validated user-safe string.

### Stores and non-React code

Use `translate` from `@ice-web-app/shared-i18n` (or the pattern already in the file):

```ts
toast.success(translate({ key: 'EMAIL_FLAGGED', ns: 'common' }));
toast.error(translate({ key: 'FAILED_TO_FETCH_THREADS', ns: 'common' }));
```

### Namespace placement

| Layer                                                                 | When                                                       |
| --------------------------------------------------------------------- | ---------------------------------------------------------- |
| `libraries/shared/i18n/locales/en/common.json`                        | Cross-zone alerts (email, chat, quick links, generic save) |
| Zone JSON (`collaboration.json`, `innovation.json`, `school.json`, …) | Feature-specific success copy                              |
| Ecosystem JSON                                                        | Portal setup / innovation-only flows (`SETUP.TOAST.*`)     |

Add keys in **en** first; update ar/de overlays when touching collaboration common keys.

### Interpolation

Use `{{name}}`, `{{label}}`, `{{groupName}}` in JSON; pass via `t('KEY', { name })`. Match Figma Title Case for product terms (Contact Card, Quick Link, etc.).

## Backend / mock API implementation

Mock API repo: `/Users/mohammad-hosein-rostami/Desktop/Desap/ice-mock-api`

### Response `message` fields

- Short, user-safe, **match English i18n** for the same user action
- Same rules 1–6 as frontend
- Update the endpoint **`schema.md`** on every message create/change (`mock-api-openapi-sync` step 5)

```python
return CardResponse(message="Your Request Cancelled", status="ok")
return SimpleStatusResponse(status="success", message="New Reminder Created")
```

### FE + BE alignment flow

1. Add or update i18n key in the correct namespace (English value = source of truth for product copy)
2. Set mock API `message=` to the same English string (with runtime interpolation where needed)
3. Wire FE with i18n-first toast — not API message fallback
4. Assert i18n copy in e2e, not raw API strings

Mobile and notification consumers may read API `message` directly — keep mock/production messages aligned even when web FE uses i18n.

## Anti-patterns

| ❌ Mistake                                                      | Why it fails                                                                            |
| --------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| One shared toast chrome for two Figma recording nodes           | 22542 started = Banner success-16 + outline check; 16228 saved = Toast success-8 + fill |
| Using `CircleCheckFill` for `ALERT_SUCCESS` / recording-started | Outline ring + green tick ≠ filled disk + white tick                                    |
| Using `CircleCheck` for recording-saved                         | Saved toast is filled success badge                                                     |
| Shipping `Successfully` on routine toasts                       | Journey doc strips it except the allow list                                             |

| Do not                                            | Do instead                                     |
| ------------------------------------------------- | ---------------------------------------------- |
| `toast.success(result?.message \|\| t(...))`      | `toast.success(t(...))`                        |
| `"Your request has been Canceled"`                | `Your Request Cancelled`                       |
| `"Changes Saved Successfully"` (routine)          | `Changes Saved` (`CHANGES_SAVED_SUCCESSFULLY`) |
| Generic `ACTION_SUCCESS_TEXT` for a known action  | Contextual key in zone/ecosystem JSON          |
| Hardcoded toast strings in TSX                    | i18n key                                       |
| `import { toast } from 'react-toastify'` in zones | `@ice-web-app/shared-utils`                    |
| Change mock `message=` without `schema.md`        | Update both in the same PR                     |

## Verification greps

Run from repo roots after toast work:

**ice-web-app** — API-first success toasts:

```bash
rg 'toast\.success\([^)]*message|result\?\.message|apiMessage' libraries apps --glob '*.{tsx,ts}'
```

**ice-web-app** — hardcoded success literals:

```bash
rg "toast\.success\(\s*['\"]" libraries apps --glob '*.{tsx,ts}'
```

**ice-web-app** — displayed `Successfully` (allow list only):

```bash
rg '": "[^"]*Successfully[^"]*"' libraries --glob '**/en/*.json'
```

**ice-mock-api** — stale grammar in messages:

```bash
rg 'message=.*(has been|have been)' app --glob '*.py'
rg 'has been|have been' app --glob '**/schema.md'
```

Expect **zero** hits except the allow-list entries (Contact Card Set up Successfully + Account
Connections “Added Successfully” / “Added to Your Card Successfully” / “Set as Default and Added…”).

## Related skills

- `mock-api-openapi-sync` — implement API + schema before FE wiring
- `figma-feature-delivery` — Figma-specific toast regression rows (keys, not obsolete English)
- `ice-calls-calendar-layout` — calendar/reminder/in-person toast keys
