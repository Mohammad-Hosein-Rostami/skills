---
name: calendar-time-grid-accuracy
description: Make time-proportional UI land on its real values — calendar day/week grids, timelines, Gantt bars, schedule blocks. Store the measurement (minutes) as the only truth and derive every label and pixel offset from it. Use when a block must start/end exactly at its time, or when a card's text disagrees with where it sits.
---

# Time-Grid Accuracy (calendar / timeline / Gantt)

Companion: `ice-calls-calendar-layout` · `figma-pixel-parity-check` · `figma-design-system-parity` · `figma-structure-before-sx`.

Applies to any UI where **position or size encodes a quantity**: calendar week/day grids, timelines, Gantt bars, availability strips, progress against a scale.

## The one rule

> Store the measurement. Derive everything else.

A timed event stores **only** `startMinutes` / `endMinutes` (minutes from midnight). Both the
displayed range string **and** the pixel offset are computed from those numbers.

```ts
// libraries/zones/collaborationZone/src/pages/iceCalls/calendar/components/calendarTime.ts
export const HOUR_HEIGHT = 80;

export const formatTimeRange = (startMinutes, endMinutes) => …;              // "09:00 am - 11:00 am"
export const minutesToOffset = (minutes, gridStartHour) => …;                // top
export const minutesToHeight = (durationMinutes) => …;                       // height
```

```tsx
top: minutesToOffset(event.startMinutes, GRID_START_HOUR);
height: minutesToHeight(event.endMinutes - event.startMinutes);
label: formatTimeRange(event.startMinutes, event.endMinutes);
```

Because the label and the geometry read the same two numbers, a card **cannot** claim
9–11 while being drawn at 8:30–10:30.

**ICE Calls day and week boards** start at **8 am** as the first gutter row
(`CALENDAR_BOARD_FIRST_HOUR` / `buildCalendarBoardHours`). Overnight hours
(**12 am–7 am**) stay in the grid **below** 11 pm — scroll down to reach them.
Do **not** put 12 am at `scrollTop: 0` and rely on scroll-to-8am; the board itself
must start at 8 am. Scroll-to-now still applies when the red indicator is present.

**Day / week board scale:** day view uses **content-aware** per-hour heights via
`dayHourHeightsFromEvents` (empty hours **60px**; grow so each timed card’s
`dayEventMinHeight` fits without clipping). Reminders keep a **fixed 46px** shell
but **stretch the hour** when the card would spill past the next hour line
(e.g. 9:30 on 60px → ~108px via `dayHourHeightForReminder`) — never fake a 15-min
duration (`68*(60/15)=272`). **Week view** uses fixed **`WEEK_HOUR_HEIGHT` = 100px**
rows (Figma `578:94927`) with **8px** cell pad on **all sides** for event shells
(`WEEK_EVENT_VERTICAL_PAD = WEEK_CELL_PAD`); empty-slot hover keeps the same inset.
Overlaps **stack** (full-width back + centered 101/117 overlay) — do **not** use
half-width side-by-side lanes; card min-heights handle density via `overflow: hidden`.
Travel-on cards: travel band is content-min above meeting start (not minute-compressed);
meeting body is minute-true start→end. Day travel-on grows meeting hours for the meeting
body min-height and the prior hour for travel chrome so the **unified** card is never
split/clipped by undersized rows; hour rules stay **under** event shells.

## Reading a time grid from Figma (this is where it goes wrong)

A Figma mock of a calendar gives you **three** things, and only two of them are authoritative:

| From Figma                                       | Authoritative? | Use it for                                              |
| ------------------------------------------------ | -------------- | ------------------------------------------------------- |
| Row height / px-per-hour                         | **Yes**        | `HOUR_HEIGHT`, column widths, gutters                   |
| Colors, radius, typography, icon, card structure | **Yes**        | DS tokens + component structure                         |
| Where a sample block sits on the canvas          | **No**         | nothing — designers place blocks by eye for density     |
| The time text inside a sample block              | **No**         | nothing — usually the same string pasted on every block |

**The mock's block position and the mock's label text are two independent pieces of
sample dressing, and they routinely contradict each other.** Never copy both into a
fixture: you would be storing the contradiction.

Instead: pick the intended time **once**, put it in `startMinutes` / `endMinutes`, and let
the label and the offset both fall out of it. If Figma's block looks like it sits at 8:30
but its label reads 09:00, that is a mock inconsistency — resolve it in favour of a real
time and say so in your summary.

## Checklist before shipping a time grid

1. **One source of truth.** No stored `timeLabel` / `scheduleTimeLabel` / `durationText`
   next to the minutes that describe the same thing.
2. **No clamped extents.** `Math.max(height, 48)` makes a short event end at the wrong
   time. Let the block be its true height and clip the content (`overflow: hidden`).
3. **Grid rules share the event coordinate space.** Draw hour rules from the same
   `n * HOUR_HEIGHT` math as the blocks (absolute rules in the positioned wrapper), not as
   per-cell `borderBottom` on `box-sizing: border-box` cells — those paint 1px early and
   live in a different space than the absolute layer.
4. **Hour labels anchor to the line they name.** A label offset into its row
   (`pt: '12px'`) makes a correctly placed block look early. Anchor the label at
   `index * HOUR_HEIGHT`.
5. **Week travel-on is one Figma card, not a clipped proportional strip.** Figma `1108:175915`
   is a single bordered unit: travel section (dots + car + label) → divider → icon well → title →
   meeting time. Anchor the absolute box at `start - travelMinutes` with
   `WEEK_IN_PERSON_TRAVEL_MIN_HEIGHT`; render {@link WeekInPersonTravelOnCard} inside — never
   squeeze the travel UI into proportional minutes (~14px for 10 min travel).
6. **Occupancy accounts for the extra range.** Hover/empty-slot detection must treat the
   travel range as occupied too, or an "Add new meeting" CTA appears under a block.
7. **Never parse a rendered label back into data.** `parseTimeRangeLabel('09:00 am - 11:00 am')`
   is a smell that display copy is being used as storage. Read the minutes.
8. **Verify one concrete case end to end.** Pick an event, compute expected `top`/`height`
   by hand, and confirm the label agrees: 9 am on an 8 am grid at 80px/hr → `top: 80`,
   2 h → `height: 160`, label `09:00 am - 11:00 am`.

## Indicator / marker chrome on a time axis (pill + connector + rule)

A time marker is usually drawn as several parts that must read as **one object**: a label pill in
the axis gutter, a connector dot on the gutter border, and a rule across the plot area. Build it
as **one flex row centred on the marker's Y** — never as independently positioned fragments.

```tsx
// ✅ One row spanning gutter + plot: parts cannot drift apart
<Box
  sx={{
    position: 'absolute',
    top,
    left: 0,
    right: 0,
    transform: 'translateY(-50%)',
    display: 'flex',
    alignItems: 'center',
  }}
>
  <Box
    sx={{
      minWidth: gutterWidth,
      display: 'flex',
      alignItems: 'center',
      justifyContent: 'space-between',
      flexShrink: 0,
    }}
  >
    <Pill />
    <Dot sx={{ mr: `${-dotSize / 2}px` }} /> {/* straddles the gutter border */}
  </Box>
  <Box sx={{ flex: 1, height: 2, bgcolor: 'error.main' }} />
</Box>
```

Three failure modes to avoid:

| ❌                                                                                          | What breaks                                                                                                                                          |
| ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Absolutely centring a variable-width pill in the gutter (`left:'50%'` + `translateX(-50%)`) | A label wider than the gutter overflows **both** edges; the left cap is clipped by the scroll container and the right end covers the connector       |
| `alignItems:'center'` on the wrapper **and** `translateY(-50%)` on the child                | Double vertical centring — the abspos child's static position is already the centre line, so the transform lifts it another half-height off the rule |
| Pill/dot in the gutter element, rule in the plot element                                    | Two coordinate spaces and two `top` values; they drift and the rule meets the pill off-centre                                                        |

Rules of thumb:

- **A variable-width label is laid out, not absolutely centred.** Anchor its left edge and let it
  grow outward; `minWidth: gutterWidth` + `space-between` puts the connector on the border for the
  common case and lets it shift for longer labels.
- **Set the centre once.** One `translateY(-50%)` on the row; children get no vertical transform.
- **A rule that encodes the same Y must be a sibling in that row**, sized with `flex: 1` — not an
  absolutely positioned box in a different parent.
- Anything wider than its gutter will be **clipped by the nearest `overflow: auto/hidden`
  ancestor**, which on a scrollable grid is the scroll container. Check the ancestor chain before
  assuming an overflowing element will be visible.

## Known failure (ICE Calls week board `25602:479678`)

Fixtures carried `timeLabel: '09:00 am - 11:00 am'` copied from the Figma mock text while
`startMinutes` / `endMinutes` were reverse-engineered from where the mock's gray blocks sat
(`8:30`–`10:30`). Every week card therefore read 9–11 and rendered half an hour early.

Fix: deleted the stored label from `CalendarWeekEvent`, `CalendarDayEvent`, and
`CalendarMonthEvent`; made minutes required; derived every label through
`formatTimeRange`; removed the `Math.max(height, 48)` clamp; moved travel time into its own
range; moved hour rules into the event coordinate space; anchored hour labels to their line.

## Known failure (time scrubber head, same board)

The red scrubber shipped with its pill absolutely centred inside the 60px hours column
(`left:'50%'` + `translateX(-50%)`) while the rule lived in the day-columns element. Results:
the pill's **left cap was clipped** by the scrolling grid container, its right end **covered the
connector dot**, and a `alignItems:'center'` wrapper combined with `translateY(-50%)` lifted the
pill and dot half a pill-height **above** the rule, so the line met the pill near its bottom edge.

Fix: one flex row spanning gutter + plot area, centred once with `translateY(-50%)`; pill anchored
left and laid out (never centre-transformed); dot straddling the border with
`mr: -dotSize/2`; rule as a `flex: 1` sibling.

## Do not

- Store a formatted time/duration string alongside the numbers it describes
- Treat a Figma sample block's y-position or its placeholder text as domain data
- Clamp the height/width of a block that encodes a duration or amount
- Draw the scale's rules in a different coordinate space from the positioned items
- Let a nested sub-block (travel, buffer, prep) consume the parent's time span
- Fabricate a default time in a mapper (`event.startMinutes ?? 9 * 60`) — make it required
- Absolutely centre a variable-width label pill inside a narrow axis gutter — it clips and overlaps
- Combine `alignItems: 'center'` with `translateY(-50%)` on the same marker — double centring
- Split a marker's pill / connector / rule across different positioned parents
