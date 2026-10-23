---
name: hide-main-sidebar
description: Hide the portal/app main navigation sidebar on a page via useGlobalStore setShowMainSidebar(false) with unmount restore. Use when a full-width page needs no left nav (calendar, attendance, My Hub, seating).
---

# Hide Main App Sidebar

Companion skill: `skill-first-governance` · store: `libraries/shared/store/globalStore.ts`.

## What this controls

| Flag              | Meaning                                                                                                       |
| ----------------- | ------------------------------------------------------------------------------------------------------------- |
| `showMainSidebar` | Primary left nav (zone menu). `setShowMainSidebar(false)` hides it and sets `sidebarWidth` to `90`.           |
| `showMiniSidebar` | Mini icon rail — separate; use `setShowMiniSidebar(false)` only when that rail must go too (Account / Setup). |

Do **not** confuse with page-local panels (ICE Calls calendar right handbar, email thread list). Those use local `useState`, not `globalStore`.

## Standard pattern (preferred)

Used by ICE Calls calendar, Attendance, My Hub, seating planners:

```tsx
import { useSettings } from '@ice-web-app/designSystem';
import { useGlobalStore } from '@ice-web-app/shared-store';
import { useEffect } from 'react';

const Page = () => {
  const { setShowMainSidebar } = useGlobalStore();
  const { updateSettings } = useSettings();

  useEffect(() => {
    updateSettings({ layout: 'vertical' });
    setShowMainSidebar(false);
    return () => {
      setShowMainSidebar(true);
    };
  }, []);

  return; /* page */
};
```

**Required:** restore `setShowMainSidebar(true)` on unmount so leaving the page brings the nav back.

## When to also touch mini sidebar / width

| Need                          | API                                                       | Example                                   |
| ----------------------------- | --------------------------------------------------------- | ----------------------------------------- |
| Hide main nav only            | `setShowMainSidebar(false)`                               | Calendar, Attendance, My Hub              |
| Hide mini rail + narrow width | `setShowMiniSidebar(false)` + `setSidebarWidth(260)`      | Account, SetupLayout                      |
| Hide both navs entirely       | `setShowMiniSidebar(false)` + `setShowMainSidebar(false)` | Navigation returns `null` when both false |

Always restore prior flags/widths in the effect cleanup.

## References

- `libraries/zones/collaborationZone/src/pages/iceCalls/calendar/index.tsx`
- `libraries/zones/defaultSchoolZone/.../takeAttendance/components/Attendance.tsx`
- `libraries/ecosystems/innovation/src/lib/portal/myHub/index.tsx`
- `libraries/shared/designSystem/src/materio/components/layout/vertical/Navigation.tsx` (`!showMiniSidebar && !showMainSidebar ? null`)
- `libraries/shared/designSystem/src/materio/@layouts/components/vertical/Navbar.tsx` — org logo when `!showMainSidebar` (ICE stays in NavHeader)

## Mistakes

- Inventing CSS/`display: none` on the nav instead of `globalStore`
- Forgetting cleanup → nav stays hidden after navigate away
- Hiding the calendar **right handbar** when the user meant the **app** sidebar (or vice versa)
- Using `setShowMiniSidebar` when only the main menu should hide
- Moving ICE + org into Navbar when main is hidden (creates a hole in the 90px rail and a large left gap). **Keep ICE in NavHeader.** Only position the org logo in Navbar: Figma header logos are ICE `x=32` + org `x=120` (gap ~32px). Rail is 90px, so org inset is **30px**, vertically centered in the 94px header. Do **not** wrap it in `width={260}` (open-sidebar leftover) or `paddingInlineStart: 4` (puts org at 106, gap ~17px, top-aligned).
- Rendering the closed-sidebar org crest from bundled `MICRO_LOGO` (`assets/images/micro-logo.png`, **126×52 @1x**). Open sidebar uses `DsImage` + `currentMicroSystem.logo` (API path via `getImageApi`) — that source is sharper on retina. Closed Navbar **must** use the same `currentMicroSystem.logo` / `DsImage` path; keep `MICRO_LOGO` only as empty-logo fallback. Do **not** point Navbar at `/api/general/file/logo` unless product requires the setup-org upload endpoint (header crest is the microsystem logo from the user session).
- **Size must match open NavHeader:** `width={150}` `height={60}` and **default `objectFit: cover`** (do not pass `objectFit: 'contain'`). API logos are often square; `contain` in a short wide box letterboxes the crest so it looks tiny vs ICE. The old **125×52** figure is the bundled MICRO_LOGO banner pixel size — not the closed-sidebar display box once the API crest is in use.
