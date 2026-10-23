---
name: ice-calls-calendar-layout
description: ICE Calls calendar page layout — two DsCards (board + handbar); hub drawer via ?drawer=calendar&view=; page tab via ?view=; plain back icon restores drawer; hide List tab; board scroll only inside grid; DsMiniCalendar; hide portal main nav.
---

# ICE Calls Calendar Layout

Companion: `skill-first-governance` · `ice-calls-jitsi` · `hide-main-sidebar` · `mui-layout-grid` · `mui-sx-theme-callback` · `figma-design-system-parity` · **`calendar-time-grid-accuracy`** (timed day/week grids).

Figma frame: Collaboration Zone calendar body **`25602:477635`** (Week + Right handbar).

## Layout contract (do not regress)

Page chrome sits on the zone/page background. The calendar body is **two separate white cards**, not one card split by a border:

```
Stack spacing={6}                    // page sections
  CalendarHeader
  Stack direction=row spacing={4}    // 16px gap between cards
    DsCard (flex 1)                  // board: toolbar + FullCalendar
      CalendarToolbar
      CalendarBoard
    DsCard (width ~385)              // right handbar — only if sidebarOpen
      DsMiniCalendar
      … holidays / My Calendar filters (later)
```

| Spec              | Value                                                                                                                                                                                                                                                                                |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Gap between cards | `spacing={4}` (16px)                                                                                                                                                                                                                                                                 |
| Board card        | `flex: 1`, `boxShadow="iceDefault"`                                                                                                                                                                                                                                                  |
| Sidebar card      | width `385` (lg), inner content ~345, padding ~20 (`p: 5`)                                                                                                                                                                                                                           |
| Dividing border   | **None** — do not use `borderLeft` on the sidebar                                                                                                                                                                                                                                    |
| Mini calendar     | **`DsMiniCalendar`** from `@ice-web-app/designSystem`                                                                                                                                                                                                                                |
| Sidebar toggle    | Exact Figma SVGs in `assets/SidebarToggleIcon.tsx` — **closed** `1100:101805` (20×16.33, `#0073E6`) · **opened** `1100:101782` (20×16.34, `#424242`). `ANCHOR: missing-ds-icon/SIDEBAR_TOGGLE_*`. Do **not** wrap in `DsIcon size` (non-square). Do **not** use live MCP asset URLs. |

## App nav sidebar (portal left menu)

While the calendar page is mounted, **hide the main app sidebar** — not the calendar right handbar.

```tsx
const { setShowMainSidebar } = useGlobalStore();
const { updateSettings } = useSettings();

useEffect(() => {
  updateSettings({ layout: 'vertical' });
  setShowMainSidebar(false);
  return () => setShowMainSidebar(true);
}, []);
```

## URL + hub drawer contract (do not regress)

| Surface                     | URL                                                                                      | Behavior                                                                          |
| --------------------------- | ---------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Calendar **page**           | `…/ice-calls/calendar?view=day\|week\|month`                                             | Board tab is the `view` search param                                              |
| Mini sidebar **Calendar**   | `…/ice-calls/calendar`                                                                   | Navigate to calendar **page** — do **not** open `DsCalendarDrawer` from mini nav  |
| ICE Calls **hub**           | `…/ice-calls?drawer=calendar&view=day\|week\|month`                                      | Opens `CalendarDrawer` on that tab                                                |
| Back from calendar page     | **No** page Back control — title only (`CalendarHeader` omits `PageHeader` `backButton`) | Soft Back + title; hub return is via main nav / hub entry, not a page Back button |
| Drawer “open full calendar” | `…/calendar?view=<tab>`                                                                  | Pass current drawer tab                                                           |
| Close drawer                | Strip `drawer` + `view` from hub search                                                  | Do not leave a stale drawer param                                                 |

Helpers: `calendar/constants.ts` — `parseCalendarBoardTab`, `iceCallsCalendarDrawerSearch`, `CALENDAR_DRAWER_PARAM_VALUE`.

**Do not** keep calendar drawer open state in hub `useState` only — URL is the source of truth so back/deep-link/restore work.

**Hub drawer card clicks (do not regress):** `CalendarDrawer` must pass the same
`onMeetingClick` / `onTaskClick` / `onReminderClick` / `onEmptySlotClick` from
`useCalendarMeetingDetails` into `DsCalendarDrawer` → `DsCalendarBoard` (all tabs).
Render `meetingDrawers` via `footer`. Omitting task/reminder handlers leaves cards
non-interactive in the compact drawer while the full calendar page still works.

**List tab:** enabled on the full calendar page toolbar (`Day/Week/Month/List`). Compact hub drawer stays day/week/month only.

**Hub drawer chrome (do not regress):** `DsCalendarDrawer` paper width **`867`**
(`DS_CALENDAR_DRAWER_WIDTH`) — override `DsDrawer size="xl"` (1000). Dialog content:
**`px: 5` (20px)** L/R, **`pt: 5`**, **`pb: 0`** (board flush to drawer bottom). Filter row =
three fields **`265`** wide, **`spacing={4}`** (16px). When overriding
`'& .MuiDrawer-paper'`, restate paper height/top/bottom (shallow merge replaces
DsDrawer paper styles).

## Recurrence + past scheduling

- Master event stores `repeatType` (`SINGLE` / `DAILY` / `WEEKLY` / `MONTHLY` / `YEARLY`).
- `GET /ice-calls/calendar` **expands** recurring masters into occurrences for `[filter_from, filter_to]`.
- FE maps those rows; board ids for non-`SINGLE` use `calendarBoardOccurrenceId`; mutations use `calendarMasterEventId`.
- Create/reschedule into the past is forbidden (API 400 + FE submit / `minDate` / empty-slot guards).
- **Edit gate:** API blocks only **completed** events (`endDateTime` < now). In-progress
  meetings (start < now ≤ end) stay editable so users can move them to a future slot.
  Do **not** treat `scheduledDateTime < now` alone as uneditable — that caused
  `"Action failed. Please try again."` when saving a reschedule of today’s afternoon seed.
- FE past-guard toast: `CANNOT_SCHEDULE_MEETINGS_IN_THE_PAST` (not generic `ACTION_FAILED`).
- `useApi` / Hey API often **throws plain `{ message }` objects**, not `Error` —
  schedule drawers must use `apiErrorMessage()` or every 400/409 collapses to
  `"Action failed. Please try again."` (conflicts never open the conflict banner).

## Mutation → refresh

After create/update/reminder: `evictIceCallsEventSyncCache()` **and** board/drawer `refresh()` via `onMutated` / `onSuccess` (page `CalendarBoard` + hub `CalendarDrawer`).

## Board scroll (page vs drawer)

- Page board lives in `DsCard` → wrap `CalendarBoard` body in `Stack flex={1} minHeight={0} overflow="hidden"` so **only the timed grid scrolls vertically**, not the page/card.
- **Responsive width (1530×920 floor):** only the **board grid** under the toolbar locks `minWidth` **991** (sidebar open) / **1392** (closed); drawer board **827** (`calendarBoardLayout`). That board scroller uses **`overflowX: auto`**. Page chrome, toolbar, sidebar, and drawer filters stay fluid — do **not** H-scroll the page or the boards row.
- Day: pin weekday header **outside** the `overflow: auto` region; scroll the hours stack.
  Page weekday header (`calendar-day-weekday-header`) needs **top + left + right** borders so the
  frame continues into the hour grid below (grid already has full border). Do not leave L/R open.
- Drawer: same flex chain (`dialogContent` + inner board scroller `flex={1} minHeight={0}`); day uses `showWeekdayHeader={false}`.
- **Page root height (required for scroll-to-now):** do **not** use only `height={1}` — Materio
  `main` grows with content (`minHeight: auto`), so `%` height never constrains the board and
  `calendar-day-scroll` / `calendar-week-board` end with `scrollHeight === clientHeight` (scroll
  is a no-op). Cap the page with viewport math, e.g.
  `height/maxHeight: calc(100dvh - 94px - 48px)` (portal header ~94 + LayoutContent pad 24+24),
  plus `minHeight: 0` and `overflowY: 'hidden'` / `overflowX: 'hidden'` on the page **at `lg+`**. Flex chain from page → `DsCalendar`
  (`alignItems: stretch`) → board card → board H-scroller → timed scroll must all keep `minHeight: 0`.
- **Below `lg` (board + sidebar stack):** unlock page height (`height: auto`, `overflowY: auto`,
  `minHeight: calc(100dvh - …)`), give the board card / scroll region a **minHeight (~560 / 480)**
  so month/day grids are not crushed under the handbar. Keep the `lg+` locked height for scroll-to-now.
- **Toolbar responsive (card width, not viewport):** `DsCalendarToolbar` uses **container queries**
  (`containerType: inline-size`, single-row floor **900px**). Below that: date + search on row 1,
  Day/Week/Month/List full-width centered on row 2 — never overlap. Do **not** rely on `md={4}`
  Grid columns (board card is often <900px while the viewport is still “desktop”).
- **Initial scroll to now:** day/week boards call `useScrollBoardToTimeIndicator` on first mount when the red
  indicator is present (today / current week). Set board `scrollTop` only — **never** `scrollIntoView`
  (that also scrolls the page). Week passes `DAY_HEADER_HEIGHT` as `contentTopOffset`. Reset key =
  today/week flag + date so navigating back to today scrolls again. Hook re-applies via timers +
  `ResizeObserver` until the board can scroll and the indicator is in the scrollport.
- E2e: assert indicator inside the **scroll box** and (for afternoon “now”) `scrollTop > 0`; also
  that the indicator’s window Y is within `innerHeight` — scroll-box-only checks pass when the
  page itself is still growing past the viewport. Width: `calendar-drawer-week-no-x-scroll.spec.ts`
  (fixed floors + H-scroll below 1530).
- Skill: **`hide-main-sidebar`**
- Same pattern: Attendance, My Hub, seating planners
- Calendar right handbar (`sidebarOpen` / `CalendarSidebar`) is independent local state

## Components

| Piece         | Location                                               |
| ------------- | ------------------------------------------------------ |
| Page          | `pages/iceCalls/calendar/index.tsx`                    |
| Toolbar       | `…/components/CalendarToolbar.tsx` (inside board card) |
| Board         | `…/components/CalendarBoard.tsx`                       |
| Sidebar       | `…/components/CalendarSidebar.tsx` (owns its `DsCard`) |
| Mini month    | `designSystem/.../calendar/DsMiniCalendar.tsx`         |
| Task CRUD     | `…/scheduleMeeting/CreateAndEditTaskDrawer.tsx`        |
| Reminder CRUD | `…/scheduleMeeting/CreateAndEditReminderDrawer.tsx`    |

## Create / Edit Reminder (Figma `578:93892`, toast `578:93951`)

**Do not** stretch `CreateAndEditEventDrawer` into Figma reminder chrome.
Reminders use a dedicated drawer (same pattern as tasks):

| Surface                     | Component / wiring                                                                |
| --------------------------- | --------------------------------------------------------------------------------- |
| Create (Add New → Reminder) | `CalendarHeader` → `CreateAndEditReminderDrawer` `mode="NEW"`                     |
| Edit (reminder card click)  | `CalendarMeetingDetailsHost` `onReminderClick` → `CreateAndEditReminderDrawer`    |
| Fields                      | Title · For Others + For Me · Date + Time · Priority + URL · Notes                |
| Field size                  | Figma inputs are **48px** → DS `size="large"` / `inputSize="large"` (medium = 44) |
| Create footer               | full-width **Confirm**                                                            |
| Edit footer                 | full-width **Save Changes**                                                       |
| Toasts                      | create `"New Reminder Created"` · edit `"Reminder Updated"`                       |
| APIs                        | `POST/PUT /ice-calls/*-event*` with `calendarKind: REMINDER` + optional `url`     |

## Create / Edit Task (Figma `578:92805` / `578:94678` / `2053:94965`)

**Do not** stretch `CreateAndEditEventDrawer` / schedule-meeting form into Figma task chrome.
Tasks use a dedicated drawer:

| Surface                         | Component / wiring                                                                                             |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Create (New Call → Create Task) | `CalendarHeader` → `CreateAndEditTaskDrawer` `mode="NEW"`                                                      |
| Edit (task card click)          | `CalendarMeetingDetailsHost` `onTaskClick` → `CreateAndEditTaskDrawer` `mode="EDIT"`                           |
| Tabs                            | Details → Assign (`DsTabs`); create: Next then Confirm; edit: Delete + Save Changes                            |
| Details field size              | Figma inputs **48px** → DS `large` (Assignees also 48). Connect-to fields are **44** → DS `medium`             |
| Status options (`578:94923`)    | `NEW` · `IN_PROGRESS` · `DONE` · `OVERDUE` (label **Over Due**) · `ON_HOLD` — mock `TaskStatus` must match     |
| Assignees (`578:94704`)         | `DsSelect` `imgUrl` photos via `toIceCallsContactSelectOption` — not initials from raw `src`                   |
| Connect dropdowns               | `type="checkbox"` + search; Figma seeds Group/Folder/Pipeline/Project A–C via `calendar_task_connections.json` |
| Create New footer               | Autocomplete `PaperComponent` footer (not a real project/folder API) — local option appended client-side       |
| Toast (`578:94920`)             | create `"New Task Created"` · edit `"Task Updated"`                                                            |
| APIs                            | `POST/PUT/DELETE /ice-calls/*-event*`, `GET /ice-calls/event/{id}`, `GET …/calendar/task-connection-options`   |

Day task cards: `data-test="calendar-day-task-card"`. Drawers: `create-task-drawer` / `edit-task-drawer`.
Figma day board seeds (`event-figma-day-task-*`) must land on **today’s calendar day** for headed e2e — roll `scheduledDateTime` when the agent clock advances.

**Figma day frame `578:95081`:** sample cards are often **hour-bucket stacked dressing**. Product day/week grids stay **time-proportional** (`calendar-time-grid-accuracy`) — do not flatten multi-hour events into unordered stacks to match the mock layout. Seed times so **local** wall-clock matches Figma labels (Iran `+03:30` ⇒ store `local−3:30` as `Z`), then restart mock (`json_cache` loads `events.json` at startup).

**Zone menu click intercept:** floating `zone-menu-center` can block Playwright clicks on day cards — use `data-test` + force/React fiber click in e2e (`calendar-day-create-edit-task.spec.ts`).

## `DsMiniCalendar` vertical rhythm (Figma ReactDatePicker `25602:477717`)

**Overall frame: `345 × 302`.** Height is the sum of parts — verify the total, not only one section:

```
302 = header 52 + gap 24 + body 226
226 = weekdays 20 + gap 6 + Dates 200
200 = pt 12 + 5×36 + pb 8
```

| Layer       | Spec                                                                      |
| ----------- | ------------------------------------------------------------------------- |
| Root        | **height 302**, width **345**, `gap: 24`, `overflow: hidden` (Figma clip) |
| Header row  | height **52** (`py: 4` + nav **44**)                                      |
| Body frame  | height **226**, `gap: 6`                                                  |
| Weekday row | height **20**, `px: 8`                                                    |
| Dates       | height **200**, **`pt: 12` / `pb: 8` / `px: 8`**, **5** week rows         |

**Known failures:**

1. Dates `minHeight: 236` (pad + **6×36**) → overall ~338 — breaks Figma **302**.
2. `height: 'auto'` after fixing clip — still wrong if Dates reserves a 6th week.
3. Treating “avoid clip” as more important than the **frame height contract** without re-checking the Figma root size.

**Fix:** use shared `DsFigmaMonthCalendar` / `buildFigmaCalendarDays` — fixed **5×7** with **always 3** leading outside days + trailing outside fill (**1** for 31-day months). Do **not** use MUI `DateCalendar` / react-datepicker Sunday-aligned weeks for mini or field dropdowns (those produce 6 lead / 5 trail for August 2026).

## `DsMiniCalendar` header (Figma Row `25602:477718`)

Figma structure (must match — do not invent from MUI defaults):

```
Row (full width, centered, py 4px)
  Cluster width 218
    IconButton 44×44  (prev)
    Current Month     (flex 1, centered) — subtitle1 / iceGray.700
    IconButton 44×44  (next)
```

Built inside `DsFigmaMonthCalendar` (`layout="mini"`).

## `DsMiniCalendar` dates grid (Figma Dates `25602:477733`)

| Spec               | Value                                              |
| ------------------ | -------------------------------------------------- |
| Dates frame height | **200** = pt 12 + **5×36** + pb 8                  |
| Day cell           | **36×36**, circle selected → `borderRadius: '50%'` |
| Horizontal pad     | **8** (`px: 2`)                                    |
| Weekday labels     | height **20**, cell width **47**                   |

**Known failures:**

1. Overriding slide to `minHeight: 180` (5×36 **without** padding) → last row / selected circle clipped.
2. Overriding slide to `minHeight: 236` (pad + **6×36**) to “never clip” → overall calendar exceeds Figma **302**.
3. `overflow: 'hidden'` with a too-small slide height → half-circle selected day.

**Fix:** Dates height **exactly 200** / 5 weeks; overall root **302**. Prefer Figma frame size over MUI’s 6-week default reservation. Skill: **`figma-structure-before-sx`**.

## Week board time grid (Figma `25602:479678`)

Events are positioned from minutes on a fixed scale. Helpers live in
`…/calendar/components/calendarTime.ts` — full workflow in **`calendar-time-grid-accuracy`**.

| Spec                     | Value                                                                   |
| ------------------------ | ----------------------------------------------------------------------- |
| Hour row height          | **`HOUR_HEIGHT` = 80**                                                  |
| Time column              | width **60**, label anchored at `index * HOUR_HEIGHT`                   |
| Day header / All-Day row | **40** / **36**                                                         |
| Day column               | `flex: 1`, min width **112**, blocks inset **4** each side              |
| Block `top`              | `minutesToOffset(startMinutes, WEEK_START_HOUR)`                        |
| Block `height`           | `minutesToHeight(endMinutes - startMinutes)` — **never clamped**        |
| Range label              | `formatTimeRange(startMinutes, endMinutes)` — **derived, never stored** |
| Travel time              | own block at `start - travelMinutes` → `start`, joined radii            |
| Hour rules               | absolute 1px rules at `n * HOUR_HEIGHT` in the day-columns wrapper      |

**Known failure:** fixtures stored `timeLabel: '09:00 am - 11:00 am'` (copied from the Figma
mock's text) _next to_ `startMinutes: 8*60+30` (reverse-engineered from where the mock's
block sat). Cards read 9–11 and rendered at 8:30. Two sources of truth for one fact.

**Fix:** minutes are the only stored time; the label is derived. See
`calendar-time-grid-accuracy` for the full checklist.

## Meeting drawer (Figma `25618:480934` in-person · `578:93432` ICE Call edit)

**Kind routing (`useCalendarMeetingDetails`):**

| Click kind          | Opens                                                    | Figma                                                                                                                                                                                   |
| ------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ice_call`          | `CreateAndEditEventDrawer` **EDIT** (full schedule form) | `578:93432` — header = meeting title; **Delete + Save Changes**; large fields; date flex + **330px** time pair (`dd MMM yyyy`); type∥repeat gap **16**; password **265** + Set password |
| `in_person_meeting` | `DsMeetingDetailsDrawer`                                 | in-person details chrome                                                                                                                                                                |
| empty slot          | `CreateAndEditEventDrawer` **NEW** ICE Call              | create schedule                                                                                                                                                                         |
| reminder card       | `CreateAndEditReminderDrawer` **EDIT**                   | `578:93656` — header **"Reminder"** (not live title); full-width **For Others** only (no For Me); **Delete + Save Changes**                                                             |
| task card           | `CreateAndEditTaskDrawer` **EDIT**                       | `578:93212` — header = task title; Details\|Assign; **Delete + Save Changes**                                                                                                           |

**Do not** open ICE Call calendar clicks on `DsMeetingDetailsDrawer` — that is the stripped
in-person chrome and fails Figma `578:93432` (missing Meeting Type / repeat / password / Delete).

Clicking an **empty slot** opens the ICE Call `CreateAndEditEventDrawer`. Hosted by
`useCalendarMeetingDetails`.

**In-person calendar meetings** stay on `DsMeetingDetailsDrawer` (live `useForm` + `DsController`).

| Field (in-person details) | Control                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Meeting Title             | `DsTextField` (full width row)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Date                      | `DsDatePicker` `dateFormat="dd MMM yyyy"`, flex grow                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Start — End               | two `DsDatePicker` `showTimeSelectOnly` in **330px** pair, 24px divider; open chevron flips; list **same width** as field (no popper gap). Time-only menu is Figma `578:96935`: `primary.opacity-16` wash, **left-aligned** 38px rows (px 20 / py 8), selected `opacity-32` + `primary.600` (not white-on-blue pill), no “Time” caption, list `hh:mm aaa`. Second field click **closes** like `DsSelect` (`DsDatePicker` controlled `open` + mousedown stop while open)                                                                                  |
| Date popper               | Fixed Figma calendar via `DsFigmaMonthCalendar` (`25602:479207` = **265×284**, **5×7**). Always **3** muted lead + trail fill (**1** for 31-day months, e.g. August 2026). Not Sunday-aligned civil weeks. `calendarContainer` replaces react-datepicker month; `ds-datepicker--figma-calendar` size/shadow overrides **must** come after base `.react-datepicker` rules. Type: month **subtitle1** · weekday **subtitle2** · day **body2** · outside **iceGray.300** · selected 36 circle **primary**. Nav **44×44**. Do **not** stretch to field width |
| Location                  | **`DsTextField`** — an editable input, not text                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Travel Time               | `InPersonTravelTimeSelect` → `travelMinutes`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |

ICE Call create/edit form rows live in `scheduleMeeting/CreateScheduleForm.tsx` (shared with hub):
large fields when `ICE_CALL` / `IN_PERSON_MEETING`; date flex + 330px times; type/repeat `spacing={4}`.

Footer save calls `PUT /ice-calls/update-event/{eventId}` then `evictIceCallsEventSyncCache` +
`onMutated` refresh. Delete uses `DELETE /ice-calls/delete-event/{eventId}` (kind-specific toast).
Date picker uses `minDate={new Date()}` — past reschedule is rejected.

**Known failure:** opening ICE Call cards into `DsMeetingDetailsDrawer` (missing Delete/Save pair,
meeting type, password) — wire `onMeetingClick` → `editEntry` + `CreateAndEditEventDrawer`.

**Known failure:** the in-person drawer shipped with `disabled` on every field (read "meeting _details_"
as read-only), `Location` rendering as an empty gray placeholder, the two-column row inset
~16px by a `Grid container` nested in a `Stack`, and `Edit` opening the ICE Call schedule
drawer. Figma showed editable inputs with floating labels and real values throughout.
See **`figma-interactive-state`** and **`mui-layout-grid`**.

## Red time indicator (Figma `25602:479731` / `25602:479740`)

**Product default: read-only current-time marker** — not draggable, not click-to-place.

- `useCalendarTimeScrubber({ interactive: false })` on day/week boards (default).
- When viewing **today** / the current week, the red line tracks **now** (60s refresh).
- On open / first mount, scroll the board to the indicator (`useScrollBoardToTimeIndicator`) so current items are in view — page + drawer.
- `DsCalendarTimeIndicator` (`data-test="calendar-time-indicator"`) — pill, connector dot, horizontal rule.
- `formatScrubberLabel` / `offsetToMinutes` / `clampMinutesToGrid` in `calendarTime.ts`.

Set `interactive: true` only for explicit scrubber UX — never on the shipped calendar day/week boards.

**Known failure:** Opening day/week at `scrollTop: 0` (8 am) while “now” is afternoon — users miss current items. Always initial-scroll to the red line when it is present.

**Event card times vs scrubber:** always render the card’s `formatTimeRange` label — **never** hide it when the red
indicator falls inside that event’s time span. The scrubber (`zIndex: 4`) draws on top; overlap is expected
(Figma week Mon 1–2 pm task + live `1:36 pm` line).

**Hour gutter labels vs scrubber:** always render static hour labels (`9 am`, `3 pm`, …) — **never** hide the label
for the hour the scrubber occupies. The red pill overlaps the gutter at the exact minute; the hour name stays at the
hour rule.

**Known failure:** `scrubber.hiddenHour` (or any guard that skips rendering the matching hour label) — users lose
orientation when the current-time line is active (e.g. `2 pm` / missing `3 pm` / `4 pm`).

**Known failure:** leaving `interactive: true` (or wiring `onColumnPointerDown` / `onHeadPointerDown`) lets users drag the red line away from real time and breaks Figma parity tests.

**Known failure:** `showTimeRange={!placement?.overlay}` (or any scrubber↔event minutes guard) — overlay and
back-layer tasks must **both** show time; do not reintroduce conditional time suppression.

## Week board layout (Figma `578:94927` / `599:117649`)

Time-proportional grid (`calendar-time-grid-accuracy`) — **not** day-style hour buckets.

| Spec               | Value                                                                                                                                                                                                                                                                                                              |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Day header height  | **38px** — selected day primary + **2px** underline (`578:94947`); time gutter **L/R/B**; **no** vertical dividers between day names; last day **R**                                                                                                                                                               |
| Day label          | Inline `{D}` + **`Stack spacing={2}` (8px)** + `{ddd}` — date `body1_500` iceGray.700 · day abbr `body2` iceGray.500 (e.g. `20 Tue`); selected → primary; **nowrap**. Never rely on a leading space in the second `DsTypography` inside a flex header — it collapses to `25Tue`. Prefer Stack gap over text spaces |
| Hour scale         | Fixed **`WEEK_HOUR_HEIGHT` = 100px** per row (`578:94927` `h-[100px]`); empty-slot hover + **event shells** use **`WEEK_EVENT_VERTICAL_PAD = 8`** (same as L/R cell pad) — do **not** stretch hours for card min-heights                                                                                           |
| Hour gutter label  | Figma `599:117752` — **60×100**, **`p: 8`**, `body2` centered (label at x/y **8**). **Not** day’s `pt: 20`. Do **not** flush labels to the hour rule.                                                                                                                                                              |
| Hour window        | **`CALENDAR_BOARD_FIRST_HOUR = 8`** → hours `[8…23, 0…7]` (8 am first; overnight at bottom). Scroll-to-now only when the red line is present — **never** open with 12 am at the top and scroll to 8 am.                                                                                                            |
| Event cards        | **Compact week variants** — 28×28 icon well, 16px Figma SVGs (`WeekTaskCard`, `WeekIceCallCard`, `WeekReminderCard`, `WeekInPersonCard`)                                                                                                                                                                           |
| Overlap            | Full-width back (`589:116099`) + **narrower centered foreground** (`595:116019`, **101/117** width) stacked on top — **not** half-width side-by-side lanes. Both stretch to real end time. Shell height = proportional minutes only — **never** `Math.max(..., WEEK_OVERLAY_CARD_MIN_HEIGHT)`                      |
| Card clipping      | Shells are time-true; card chrome may clip via `overflow: hidden` when content is taller than the shell. Do **not** grow the shell past `endMinutes`                                                                                                                                                               |
| Reminders          | Fixed **46px** at start time — never stretch (`614:117375`); checkbox **orange only** (never green success)                                                                                                                                                                                                        |
| In-person travel   | **Week** (`589:115565`) + **Day** (`589:115354`): travel content-hug above start; **green divider on the hour-rule axis** via `weekAbsoluteTop` (exact 11:00 — **not** inset `weekInsetTop` +8px). Meeting body spans start→end on that axis.                                                                      |
| Empty slot hover   | Full cell gray + centered `ADD_NEW_MEETING` chip — inset **8px** inside hour cell (`578:94927` p-[8px]); hour rules at `zIndex: 0`, events above                                                                                                                                                                   |
| Event card hover   | Figma `Nicky Created` shadow: `0px 1px 8px rgba(66,66,66,0.18)` (`WEEK_EVENT_HOVER_SHADOW`); raise z-index so shadow clears neighbors                                                                                                                                                                              |
| Event stack gap    | Prefer **minutes-true shells** — `weekEventVerticalGaps` is a no-op (gap was pulling timed shells off hour lines). Overlap stacking (full + centered overlay) separates concurrent events                                                                                                                          |
| Card internals     | Icon block px **8** py **4** · icon→text **2px** · title→time **8px** · text pb **8px** (`589:115497`)                                                                                                                                                                                                             |
| Task time wrap     | `WeekTaskCard` time always `pre-line` + break after `-` (same as overlay `595:116019`). **Never** `whiteSpace: nowrap` on non-overlay — hub drawer week columns clip mid-`am` (`09:00 am – 11:00 o`). Title already `wordBreak: break-word`                                                                        |
| All-day row        | **Removed** — not in Figma `578:94927`                                                                                                                                                                                                                                                                             |
| Sidebar toggle     | Toolbar `data-test="calendar-sidebar-toggle"` — open `578:94927`, hidden `599:117649`                                                                                                                                                                                                                              |
| Day column borders | Per-column `borderRight` gray-16 on timed grid (`calendar-day-column`); time gutter header + grid column **`borderLeft`** (continuous left edge); board wrapper **`borderTop`** (`578:94946`)                                                                                                                      |

**Known failure:** Stretching week hour rows via `bumpHoursForBlock` / `weekHourHeights` — Figma uses fixed **100px** rows; overlap + `overflow: hidden` handle short spans instead.

**Known failure:** Clamping week shell height with `Math.max(proportional, WEEK_OVERLAY_CARD_MIN_HEIGHT)` / `WEEK_IN_PERSON_TRAVEL_MIN_HEIGHT` — a 4–5 pm overlay spills past 5 pm; travel cards spill past end. Shells must be minutes-only.

**Known failure:** `WeekTaskCard` time `whiteSpace: overlay ? 'pre-line' : 'nowrap'` — drawer/narrow week columns truncate the range. Always wrap after `-` like Figma overlay.

**Known failure:** Putting **12 am** as the first gutter row and scrolling to 8 am — product boards must **start at 8 am**; overnight (12 am–7 am) belongs **below** 11 pm. Also wrong: deleting overnight rows entirely (`START_HOUR = 8` with only 8–23).

**Known failure:** hour rules at `zIndex: 1` (or above events) — grid lines paint over cards and clip borders/text. Keep rules at `zIndex: 0`; events ≥ 1. Use `weekInsetTop` / `weekInsetHeight` with **`WEEK_EVENT_VERTICAL_PAD = 8`** (match L/R cell pad) + horizontal `WEEK_CELL_PAD` 8.

**Known failure:** Half-width side-by-side lanes for overlaps — Figma stacks a **full-width back** under a **centered 101/117 overlay**. Side-by-side makes travel cards unreadable in narrow drawer columns and is not the product overlap model.

**Known failure:** Day reminder `py: 0.5` when `fillSlot` — Figma `585:117453` is **`py-[8px]`** always. Shell also needs **`DAY_EVENT_VERTICAL_PAD = 8`** so top/bottom match L/R cell pad.

**E2E:** `apps/e2e/agent-browser/tests/flows/calendar-week-figma-parity.spec.ts` · `calendar-drawer-week-task-time-wrap.spec.ts` · pixel captures `capture-calendar-week-pixel-baselines.mjs`.

## List board layout (Figma `599:118140` sidebar closed · `2391:98544` sidebar open)

Agenda view — grouped by day, no timed grid.

| Spec                | Value                                                                                                                                            |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Day header height   | **48px** — `primary[20]` bg (`2391:98565` / `2391:98579`)                                                                                        |
| Selected day header | **4px** left `primary.main` + `body1_500` **`primary.main`** — toolbar `currentDate` only                                                        |
| Other day headers   | gray-16 **bottom only** · `body1_500` **`iceGray.700`** — **no** primary left accent · **no** L/R (card already frames the board)                |
| Event row height    | **38px** (tasks / meetings / ICE) · **42px** reminders (`599:118170`)                                                                            |
| Row padding         | **16px** horizontal · **8px** vertical (`599:118161`)                                                                                            |
| Row gap             | **16px** between item block and time column                                                                                                      |
| Row borders         | **1px** `#ececec` **bottom only** (`2391:98589` stacked in the card). Do **not** add L/R on rows **and** the board/card — that doubles the rail. |
| Item layout         | Accent bar **4×16** (Figma `Line 1482` stroke-width 4, round caps) + 6px gap + `body1` title (`478:96109` / `2391:98589`)                        |
| Reminder item       | Unchecked **warning** orange border · checked **success** green fill (`478:96119` / `478:96137`) — time stays visible when completed             |
| Time column         | **156px** trailing `body2` left-aligned in column (`599:118163`)                                                                                 |
| Leading time rows   | **178px** `body1` before title (`599:118202`) — seed ids `*-list-leading-*`                                                                      |
| Time format         | Range `09:00 am - 11:00 am`; reminder single `07:00 pm`                                                                                          |
| Scroll              | `calendar-list-board` scrolls inside board card only — reset `scrollTop` on date change; **never** `scrollIntoView` (page jump)                  |
| Sidebar toggle      | **Open** `2391:98544` (~991 board) · **Closed** `599:118140` (~1392)                                                                             |

**Components:** `DsCalendarListView` · `ListEventRow` — no type badges/chips; accent bar only.

**Data:** `GET /ice-calls/calendar` via `calendarRangeForView('list')` — **`filter_from` = start of selected day**, **`filter_to` = end of selected + `LIST_AGENDA_DAYS_AFTER` (7) days**. `mapCalendarEntriesToListEvents()` keeps that same window. Always render the selected-day header (empty if no events). Mini-calendar month chevrons call `onChange` (same day clamped) so the list refetches. Figma pixel seeds: `event-figma-list-jan*` on **14 January 2022** (navigate toolbar/mini-calendar to that date; Jan 14–21 window covers Jan 14–20 seeds).

**Known failure:** List board **and** each Days row both paint `borderLeft`/`borderRight` — the card already frames the board, so item L/R doubles the rail. Rows/headers: **bottom only**; selected header keeps the **4px** primary left accent.

**Known failure:** List accent as **2×16** filled rect — Figma `Line 1482` is **stroke-width 4** with round caps. Use **4×16** + `borderRadius: '2px'` (string — bare `2` is MUI theme units → ~12px).

**Known failure:** Dumping the full month grid from ISO-week start and only highlighting selected day — changing the day looks like a no-op because July/early-month rows stay at the top and empty selected days have no section. List must start at `currentDate` through +7 days.

**Known failure:** `scrollIntoView` on the selected header — scrolls the **page** down as well as the board. Use board `scrollTop = 0` only.

**E2E:** `apps/e2e/agent-browser/tests/flows/calendar-list-figma-parity.spec.ts` · `calendar-list-date-change.spec.ts` · pixel captures `capture-calendar-list-pixel-baselines.mjs`.

**Known failure:** Reusing day cards (`DayTaskCard` / 36px wells) in week columns — too wide/tall for Figma `589:115497`.

**Known failure:** Icon well with `box-sizing:border-box` + `p:6` + `1px border` — content area shrinks to ~14px and **clips** 16px Figma glyphs. Center icons in a fixed 16×16 / flex well without padding shrink.

**Known failure:** Dense mock seeds (day + week on same Sunday hour) — prefer sparse times that still show every card type; Figma sample times are dressing, not required.

**Known failure:** Replacing `DsCalendarEmptySlot` chip + full-cell hover with an `inset: 0` absolute button and `flex: 1` only — week absolute hour wrappers collapse to 0 px, CTA text pins to the hour rule; day chip shape disappears.

**Known failure:** Day in-person card with **title above** icon+time row — Figma day (`25601:476923` / `599:117933`) is **horizontal**: icon **36** + gap **12** + column (title `body2_500` / time `subtitle2`). Do not invent a second “full span” time under the meeting range.

**Known failure:** Week in-person travel/title/time `noWrap` / ellipsis — Figma week + long-text frames wrap. Use `whiteSpace: normal` / `pre-line` (break after `-` like week tasks).

**Known failure:** Compressing day/week/month **board grids** below the 1530 Figma floors (page **991** / **1392**, drawer **827**) — columns become unreadably narrow. Lock `minWidth` on the board inner via `calendarBoardLayout` and **H-scroll only the board scroller** — never the page or boards row.

**Hour grid rules (day `578:95082` / week `578:94927`):** draw horizontal rules as an **absolute overlay** so lines span the **60px time column and all day columns**. Day: paint rules **inside** `calendar-day-timed-column` with `left: -TIME_COL_WIDTH` (above the paper bg) — sibling rules under the timed column are covered and vanish from the board. Week: full-width overlay on the timed grid stack. Do **not** render rules only inside a day-columns flex child that stops at the gutter edge.

**Day hour gutter labels (drawer `25601:476916` / stretched `25601:476920`):** each cell is **60px** wide with label **`pt: 20`** + `body2` centered — **not** vertically centered in the hour, and **not** flush to the hour rule. Stretched hours keep the same `pt: 20` (label stays near the top).

**Week hour gutter labels (`599:117752`):** **60×100**, **`p: 8`**, `body2` centered — **not** day’s `pt: 20`.

**Known failure:** Day hour rules as Stack siblings _before_ `calendar-day-timed-column` (`bgcolor: paper`, `zIndex: 1`) — board looks borderless while the gutter still shows lines. Keep rules inside the timed column (extend left into the gutter).

**Known failure:** Day hour labels at `top: hourTops[i]` with no padding — labels sit on the line / look mid-cell when rows stretch.

## Month board layout (Figma `578:92457` / sidebar closed `599:117449`)

Month view is a **Mon–Sun grid** inside the calendar board card (sidebar open/closed is page-level via `calendar-sidebar-toggle` + filter list).

**Static cells — do not flex-stretch.** Day cells are locked to Figma height **125px** (`578:92494` / `599:117486`). The month board uses `overflow: auto` and scrolls when the card is shorter than `38 + weeks×125`. Never use `flex: 1` on week rows to fill leftover card height — that produced ~142px cells and squashed last rows.

| Spec                        | Value                                                                                                                                                                                                                                                                   |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Weekday header row          | **38px** height · **8px** pad · `body1` gray-500 centered · **scrolls with board** (not sticky) · **first col** `borderLeft` · **last col** `borderRight` only — **no** bottom border · **no** vertical between middle days · board frame **`borderTop`** (`578:92476`) |
| Day cell                    | **125px** height (static) · **8px** pad · **10px** gap · **`gray-16`** (`#ECECEC`) **`borderTop` + `borderRight`** per cell · **first col** also `borderLeft` · **no** outer board border · **no** `borderBottom` on cells                                              |
| Column width                | Equal share of board card width — **~141.57** when sidebar open (board **991**), **~198.86** when closed (board **1392**)                                                                                                                                               |
| Selected day                | Cell bg `iceGray opacity-16` · day number in **20px** primary circle (not 28px)                                                                                                                                                                                         |
| In-month day number         | `body1` gray-500 · outside month gray-300                                                                                                                                                                                                                               |
| Event rows                  | **26px** height · **8px** left / **2px** right pad · **2px** radius                                                                                                                                                                                                     |
| Task / ICE call / in-person | **4×16px** accent bar, round caps (primary / gray-700 / success) · **6px** gap · title · **10px** time right (`formatTimeOfDay`)                                                                                                                                        |
| Reminder                    | **18px** orange checkbox (checked = fill + tick) · **4px** gap · title + time                                                                                                                                                                                           |
| Overflow                    | Show max **3** rows then `+N more` i18n key `CALENDAR_N_MORE`                                                                                                                                                                                                           |

**Month item Figma nodes:** task `585:117559` · ICE call `585:117617` · in-person `1097:102253` · reminder `585:117463`.

**E2e:** `apps/e2e/agent-browser/tests/flows/calendar-month-figma-parity.spec.ts` · board `data-test="calendar-month-board"` · cells `calendar-month-cell` / `calendar-month-empty-cell` · items `calendar-month-{task,ice-call,in-person,reminder}-item`. Assert **125px** height with sidebar open **and** closed.

**Known failure:** Flex-stretching week rows (`flex: 1` / `minHeight: 0` fill) — cells grow past 125px (~142px) and the last week row squashes. Lock `MONTH_CELL_HEIGHT = 125` and scroll the board.

**Grid line color:** `calendarGridLineColor` → `iceGray.opacity-16` (`#ECECEC`) — Figma `gray-16` on week `578:94927` and month `578:92457`. Do **not** use `iceGray.200` for calendar grid dividers.

**Known failure:** Week view absolute `calendar-day-column-rule` overlays — replaced by per-column `borderRight` on `calendar-day-column`; overlays double-stacked lines with cell borders.

**Known failure:** Month cells with `borderBottom` + `last-of-type borderRight: none` — Figma uses `borderTop` + `borderRight` only (rows stack via top border); last column keeps `borderRight`.

**Known failure:** Using 28px selected-day circle or week-sized card components inside month cells — month rows are compact 26px chips, not hour cards.

**Known failure:** Empty month cell absolute `inset: 0` slot capturing clicks on day numbers / neighboring cells — set slot `pointerEvents: 'none'` and CTA `pointerEvents: 'auto'`; click the **cell** (not the day-number text) in e2e.

## Month sidebar (Figma `590:115592` / header `590:115597` / items `590:115593`)

When `view === 'month'`, **hide** `DsMiniCalendar` (`data-test="calendar-mini-calendar"`) and render `DsCalendarMonthSidebar` instead.

| Spec       | Value                                                                                                                |
| ---------- | -------------------------------------------------------------------------------------------------------------------- |
| Header     | **84px** · border-bottom gray-200 · `Overview {D} {Month}` (`CALENDAR_DAY_OVERVIEW`) · **20px** error-16 count badge |
| Event list | **8px** gap · day-style cards (task chips + time, in-person, reminder checkbox)                                      |
| Below list | Divider · `DsCalendarFilterList` (“My Calendar”) unchanged                                                           |
| Data       | `monthEventsForSelectedDay(monthEvents, currentDate, enabledFilters)` — cell click updates `currentDate`             |

Test ids: `calendar-month-sidebar` · `calendar-month-sidebar-header` · `calendar-month-sidebar-events` · `calendar-month-sidebar-{type}-item`.

**Known failure:** Leaving mini calendar visible on month tab — sidebar must switch to selected-day overview panel.

**Known failure:** Empty month cell with `DsCalendarEmptySlot` as flex child only — hover gray misses the day-number band. Use absolute `inset: 0` empty slot under the day number, `pointerEvents: 'none'` on the slot + `auto` on the CTA, and cell-level `&:hover` for full-cell gray.

**Known failure:** Completed month reminders with `line-through` or time label — Figma shows checked box + title only, no strikethrough, no time.

## Day board layout (Figma `578:95082` + product time grid)

Day view is **time-proportional** (same contract as week — `calendar-time-grid-accuracy`), **not** an hour-bucket stack:

- Absolute shells: `top` / `height` from `startMinutes`→`endMinutes` via `weekInsetTop` / `weekInsetHeight` with **`DAY_EVENT_VERTICAL_PAD = 8`** (same as L/R `DAY_CELL_PAD`) + horizontal pad 8. Hour heights from **`dayHourHeightsFromEvents`** (empty **60px**; grow so `dayEventMinHeight` fits). **Reminders** stay **46px** tall but stretch the hour via `dayHourHeightForReminder` when they’d cross the next hour line (9:30 → ~108px) — never treat as 15-min for scale. Board hours **`[8…23, 0…7]`** via `buildCalendarBoardHours` — first row **8 am**; overnight at bottom. Scroll-to-now when the red line is present.
- Travel-on in-person: travel chrome (content-min band) sits **above** meeting start; green divider on start; meeting body spans `start` → `end` only.
- Reminders: fixed **46px** at start with Figma **`py: 8`** (`585:117453`) — do not shrink py when `fillSlot`; do not inflate the hour.
- Overlaps + hover: full-width back + narrower centered overlay (`dayEventPlacements` / `DAY_OVERLAY_WIDTH_RATIO`) + **`WEEK_EVENT_HOVER_SHADOW`** on hover — same as week. **Never** half-width side-by-side lanes.
- Empty hours: `DsCalendarEmptySlot` absolute per hour when no event intersects that hour.
- Cards `fillSlot` → `height: 100%` inside the shell; grow hours for min content — **never** clip day cards or invent empty flex inside the shell.
- Page + hub drawer share `DsCalendarDayView`.

**Known failure:** Bucket multi-hour events into the start-hour cell (`eventsByHour` + `dayHourRowMinHeight`) — a 9–11 task fills only 9–10; a 9:00–9:30 call fills the whole hour. Always derive geometry from minutes.

**Known failure:** `dayHourHeight` + fake 15-min reminder ranges → 12 pm row ~272px with a 46px reminder. Use `dayHourHeightsFromEvents` + `dayHourHeightForReminder` (fixed card fit, not duration scale). Skipping reminder stretch entirely leaves mid-hour reminders (e.g. 9:30) straddling the next hour border on a 60px row.

**Known failure:** 8px vertical cell pad + hour rules above shells → visible grid borders around cards. Day vertical pad is **0**; event `zIndex` above rules.

**Known failure:** absolute `top`/`height` without overlay lanes for same-window events — use `dayEventPlacements` (not hour buckets) when task 9–11 + ICE overlap.

Download exact SVGs from Figma MCP asset URLs into `calendar/assets/svg/`, then generate TSX with **original hex fills preserved** (`#F5F5F5`, `#424242`, `#28A745`, etc.) — never blanket `currentColor` on multi-tone icons.

| Icon              | Filter node  | Day-board node |
| ----------------- | ------------ | -------------- |
| ICE Calls camera  | `944:127245` | `585:117589`   |
| In-person meeting | `947:127472` | `589:115354`   |
| Task clipboard    | `944:127437` | `589:115371`   |
| Reminder doc      | `948:101810` | —              |
| Travel car        | —            | `589:115354`   |

Each component needs `// ANCHOR: missing-ds-icon/<KEY>` until promoted to `IceIconsEnum` + `DsIcon`. Filter icons use filter SVGs; day cards use larger day variants (`DayIceCallCameraIcon`, `DayInPersonIcon`, `DayTaskIcon`).

**Known failure:** wrapping Figma SVG icons in a parent `Box` with `color: iconColor(theme)` — all paths inherit one color and lose Figma parity.

## Calendar header controls (Figma)

| Control                  | Node        | Implementation                                                                                                                                                                 |
| ------------------------ | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Calendar source dropdown | `578:96759` | `CalendarSourceSelect` — paper **265×208** (`py: 9px`), primary-16 list, selected row primary-32 + primary text, **38px** rows, “Sync New Calendar” as + action                |
| New Call split button    | `578:95172` | Primary **Add New** (44×44, r8, shadow-xs) → schedule ICE Call drawer; chevron menu: ICE Call, In-person Meeting, **Task**, **Reminder**                                       |
| My Calendar filters      | sidebar     | `DsCalendarFilterList` — Figma SVG icons with **preserved multi-color fills** (do not set `color` on the 32×32 well — flattens icon); colored **square** custom checkbox icons |

`DsDropDownGroupButton`: menu paper **182×168** (`py: '8px'`, `MenuList` `py: 0`), item height **38**, row/item bg primary-16, hover/selected primary-32; `persistSelection={false}` so the first item is not always shown as selected.

## In-person meeting drawer (Figma `1103:111704` + travel list `1108:110123` + toast `1105:108107`)

Use `CreateAndEditEventDrawer` with `calendarKind="IN_PERSON_MEETING"` — **not** the task drawer.

Field order: **Meeting Title → Date/Time → Required Attendance (Optional) → Location + Travel Time**.

- Fields are **large / 48px**. Date is `dd MMM yyyy` with calendar + **chevron** (no clear X). Times use clock + chevron. Date/time row: date flex + **330px** time pair, **8px** gap. Location/Travel **16px** gap. Body stack gap **24px**. Body padding **20px**.
- Travel Time is a **select** — Figma `1108:110123` copy, mapped to `travelMinutes`:
  - `None` → `null` (no travel strip)
  - `At Start of Travel Time` → `0`
  - `15 minutes before Travel Time` → `15`
  - `30 minutes before Travel Time` → `30`
  - `1 hour before Travel Time` → `60`
  - `2 hour before Travel Time` → `120` (singular “hour” is exact Figma)
- Reuse `InPersonTravelTimeSelect` (instance `renderOption` body1 / body1_500) — **do not** change global `DsSelect` list typography.
- Title copy: `"Create In Person Meeting"`; placeholder `"New Meeting"`; location `"e.g., Conference Room B"`.
- Success toast (create **and** edit, including `DsMeetingDetailsDrawer` save): `"New In Person Meeting Set"` (`NEW_IN_PERSON_MEETING_SET_SUCCESSFULLY`). Do not fall back to generic `LABEL_UPDATED_SUCCESSFULLY` / Event.
- Prefill date/time on open (`buildNewFormDefaults`) like the task drawer.
- Required Attendance / Assignees / For Others person options: `toIceCallsContactSelectOption` → `avatar.imgUrl = profilePicture`. Initials-only circles mean the row used `src` (portal 404) instead of `getImageApi`.

E2E: drawer content portals outside the wrapper `Box` — assert `getByRole('heading', { name: '…', exact: true })`, not the hidden wrapper `data-test`. Day in-person cards: `calendar-day-in-person-card`.

**Known failure:** treating Travel Time as a duration list (`15/30/45/60 minutes`) instead of the Figma reminder-style labels. Board travel strips still use `travelMinutes > 0`.

## Figma day seeds + e2e hygiene

- `events.json` Figma day rows only for board parity tests (`event-figma-day-*` on **2026-08-23**).
- Create-flow e2e must **delete** created rows via `DELETE /ice-calls/delete-event/{eventId}` (Bearer cookie) after asserting toast — do not leave `E2E task …` pollution in `events.json`.
- Restart mock API after hand-editing `events.json` (`json_cache` loads at startup).
- Portal `VITE_MOCK_API_URL` must be `https://ice-mock-api.desap.net` (deployed mock only — no machine-local mock URL).

## Do not

- Change global `DsDrawer` / `MuiDrawer` theme for calendar drawer chrome — use instance props: `hideShadow` (no paper cast) and existing `hideBackdrop` (no dim). Calendar schedule drawers pass **`hideShadow`** only (keep backdrop)
- Stretch the date calendar to the Date field width — Figma `25602:479207` is a fixed ~265px panel (`ds-datepicker--figma-calendar`); only time-only lists match field width (`ds-datepicker--match-field-width`)
- Put `ds-datepicker--figma-calendar` overrides **before** base `.react-datepicker` rules — base padding/nav/body1 days win and you get ~320px with 24px chevrons and 16px day type even when width is 265
- Expect every month to be exactly **284** tall — Figma frame is **5** week rows; 6-week months (e.g. Aug 2026) correctly grow to ~320. Parity-check against a 5-week month (e.g. July 2026 / Jan 2025)
- Do not ship the default white/centered/blue-pill time menu; Figma `578:96935` is a full-row tinted list
- Disable the meeting drawer's fields, or treat it as a read-only preview
- Wire its `Edit` action to `CreateAndEditEventDrawer` — different entity, different endpoint
- Render `Location` as text — it is a `DsTextField`
- Nest a `Grid container` in the drawer's `Stack` — the field row will be inset by the gutter
- Store a `timeLabel` next to `startMinutes` / `endMinutes` — derive it (`calendar-time-grid-accuracy`)
- Store the scrubber's label string — derive it with `formatScrubberLabel(minutes)`
- Enable scrubber drag/click on day/week boards (`interactive: true`) — product ships read-only current time only
- Copy a Figma sample block's y-position **and** its placeholder time text into a fixture
- Clamp a block's height (`Math.max(height, 48)`) — it then ends at the wrong time
- Nest the travel-time strip inside the meeting block's own span
- Draw hour rules as per-cell `borderBottom` while events are absolutely positioned
- Wrap board + sidebar in a **single** `DsCard` with an internal `borderLeft` splitter
- Re-create a zone-local `MiniMonthCalendar` — use `DsMiniCalendar`
- Use `borderRadius: '500px'` for the selected day circle — use `'50%'` (skill `mui-sx-theme-callback`)
- Put FullCalendar / event list UI into the sidebar card
- Fight MUI header DOM with CSS when Figma layer order differs — replace the slot
- Shrink slide `minHeight` below Figma Dates content **including padding**, or clip with `overflow: hidden`
- Inflate Dates for MUI’s 6-week default (`236`) when Figma overall height is **302** (Dates = **200** / 5 weeks)
- Use MUI/react-datepicker Sunday-aligned weeks for mini/field calendars — August 2026 becomes **6** lead / **5** trail; Figma requires fixed **3** lead + trail fill via `DsFigmaMonthCalendar`
- Leave the portal main nav visible on this page — hide via `setShowMainSidebar(false)` + cleanup (`hide-main-sidebar`)
- Fall back to day/week/month **fake fixture** files — boards map `GET /ice-calls/calendar` only
- Leave `useIceCallsCalendar` as a stub or omit `events` props on day/week/month views
- Ship `DEMO_HOLIDAYS` — holidays only from API props (hidden when empty)
- Use exact-key `evictApiCache` after schedule mutations — use **`evictIceCallsEventSyncCache`** so all range queries refetch
- Put **12 am** at the top of day/week and “fix” it with scroll-to-8am — boards must **start at 8 am**; overnight goes **below** 11 pm
- Delete overnight rows (8–23 only) — keep full 24h wrapped order `[8…23, 0…7]`
- Open today/current week stuck away from the red “now” line — still scroll to the indicator when present

- Stack day cards into the start-hour row only (ignore end time / duration) — day shells must stretch `startMinutes`→`endMinutes` (page + drawer)
- Force day in-person+travel card `minHeight: 168` + `flex:1` body — invents empty green; timed shell owns height, card `fillSlot` fills it
- Leave padding on empty hour cells (`pl`/`pt`) — hover fill will not reach cell edges
- Set `color` on filter icon wells when using multi-color Figma SVGs — icons flatten to one tone
- Keep calendar drawer open in hub `useState` only — use `?drawer=calendar&view=` so back from the full page restores the drawer on the correct tab
- Soft / filled back control on the calendar page — plain text/icon arrow; navigate with `iceCallsCalendarDrawerSearch(view)`
- Let the calendar **page** grow a second scrollbar — constrain page height with
  `calc(100dvh - 94px - 48px)` (not bare `height={1}`), keep `CalendarBoard` on
  `flex={1} minHeight={0} overflow="hidden"`; only the timed grid scrolls (drawer already does this)
- Open day/week at `scrollTop: 0` while “now” is afternoon — page must be height-capped so the
  board overflows; then `useScrollBoardToTimeIndicator` can place the red line in view

- Ship the List toolbar tab without product sign-off — List is enabled on the full page; hub drawer remains day/week/month
- Treat `repeatType` as display-only on the seed row — `GET /calendar` expands occurrences; FE must not invent client-only RRULE grids
- Call `evictIceCallsEventSyncCache` without refreshing an active board — also pass `onSuccess` / `onMutated` → `refresh()` on page + drawer
- Allow create/reschedule into the past — API rejects; FE must guard submit, `minDate`, and empty-slot clicks
- Treat in-progress meetings as uneditable because `scheduledDateTime < now` — only **ended** events are locked; otherwise reschedule saves fail with a generic Action failed toast
- Omit `onReminderToggle` on the hub calendar drawer — same mutation + eviction path as the page board
- Mutate recurring board rows with the occurrence composite id — strip via `calendarMasterEventId` before `update-event`
- Pass `src: profilePicture` into calendar person `DsSelect`s (Required Attendance, Assignees, For Others) — use `imgUrl` / `toIceCallsContactSelectOption`
- Use `type="multi"` for Calendar Items in the hub drawer — use **`type="checkbox"`** (Figma `25573:485475` / field `25601:476900`) with `limitTags={2}` + `disableClearable`
- Stretch Calendar Items past Figma **265×44** — lock `sx={{ width: 265, maxWidth: 265 }}` on the drawer select
- Put `overflowX: auto` on the **outlined field root** — that scrolls chips under the L pad / chevron and clips the first chip. Field root is **`overflow: hidden`**; only `[data-test="ds-select-checkbox-tags-scroll"]` scrolls; `+N` stays outside that scroller
- Let checkbox multi-select **wrap chips / grow height** (`+2` on a second line) — Figma field is **fixed 44px**, `flexWrap: nowrap`, **horizontal** scroll inside tags only; `limitTags` + `+N` must stay on the **same** row (custom `renderTags` while focused too)
- Paint selected checkbox options in **primary** text / heavier weight — Figma labels are always **body2 / iceGray.700**; only the checkbox glyph shows selected state

- Use theme default Checkbox icons in checkbox `DsSelect` options — unchecked Figma stroke is **primary.500** 18×18 r4, not gray `text-secondary`; selected option **label stays iceGray.700** (checkbox carries state — do not paint option text primary)
- Sticky month weekday header (`position: sticky`) — event chips share z-index and bleed through Mon–Sun labels while scrolling; header scrolls with the board
- Reset drawer `currentDate` on every `open===true` effect — only reset on **closed→open**; tab changes must keep the selected day
- Leave month empty-cell CTA `pointer-events: auto` while opacity 0 — it steals clicks so the Date field / selection never updates; enable CTA pointer-events on hover only, and call `onDateChange` before `onEmptySlotClick`
- Wire only `onMeetingClick` on hub `CalendarDrawer` — task/reminder cards look clickable but do nothing; pass `onTaskClick` / `onReminderClick` through `DsCalendarDrawer` like the full page board

## Shared calendar later

Zone page is the source of truth while building. Reuse into `@ice-web-app/shared-ui` calendar only when the page is stable — do not diverge board/sidebar layout when promoting.
