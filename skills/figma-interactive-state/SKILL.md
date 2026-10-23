---
name: figma-interactive-state
description: Read a control's interactive state (enabled / disabled / readonly / error) from Figma's actual fills and text tokens instead of inferring it from what you assume the screen is for. Use when implementing forms, drawers, detail/preview screens, or any frame containing inputs, selects, and buttons.
---

# Reading Interactive State From Figma

Companion: `figma-pixel-parity-check` · `figma-design-system-parity` · `ds-typography-first` · `skill-first-governance`.

`disabled` is a **design-system variant with observable tokens**, not an inference from the
screen's purpose. Never decide a field is disabled because the screen "looks like a detail
view", "is a preview", or "has an Edit button".

## Read the tokens, not the intent

| Observable in Figma                                                                                                      | State        |
| ------------------------------------------------------------------------------------------------------------------------ | ------------ |
| White / `background.paper` fill, `iceGray.700` value text, **label floated onto the border**, visible chevron/affordance | **enabled**  |
| Gray fill (`iceGray['opacity-8']`-ish), muted value text, label sitting **inside** as placeholder, no affordance         | **disabled** |
| Normal fill but no caret/affordance, value styled like body text                                                         | **readonly** |
| `error.main` border + helper text                                                                                        | **error**    |

The floating label is the strongest single tell: MUI floats the label onto the border only when
the field has a value or focus, and a **disabled empty** field instead shows its label as gray
placeholder text inside the box. If the Figma frame shows `Location` floated on the border with
`Conference Room B` below it, that field is an **enabled input holding a value** — not a
disabled box.

## Before setting `disabled`, answer these

1. Does the Figma frame actually show DS disabled tokens for this control? Quote the fill.
2. Is there a **state variant** in the file (default / hover / disabled / error) — did I check
   all of them, or only the one frame I was handed?
3. Which API owns this entity, and does the user have permission to edit it here? A missing
   backend endpoint is **not** a reason to disable the UI — build the inputs and leave the
   submit wired to a `TODO`.

Disabling everything is never the safe default. It silently removes functionality and reads as
a bug to the user, while a live input with an unwired submit reads as work in progress.

## Do not hand off to a drawer that targets another API

An "Edit" affordance must edit the entity you are looking at, through **that entity's own API**.

Before reusing an existing drawer/form for a different entity, confirm they share an entity and
endpoint. If they don't, edit in place in the owning surface instead.

```
Calendar meeting  → calendar API   → edited in place in MeetingDetailsDrawer
ICE Call event    → /ice-calls/*   → CreateAndEditEventDrawer
```

Reusing the ICE Call schedule drawer to "edit" a calendar meeting looks like reuse but binds
the wrong payload to the wrong endpoint, and drops fields the other entity doesn't have.

## Known failure (ICE Calls calendar meeting drawer `25618:480934`)

Figma showed a normal editable form: white fills, floating labels, real values, chevrons on the
date/time/travel controls. I read "meeting **details**" as "read-only preview" and set
`disabled` on every field, so the drawer shipped fully greyed out with `Location` and
`Travel Time` rendering as empty gray placeholders. The footer `Edit` button then opened the ICE
Call **Schedule New Meeting** drawer — a different entity on a different endpoint.

Fix: fields are live `DsController` inputs over `useForm`, `Location` is an editable
`DsTextField`, `Travel Time` an editable `DsSelect`, and the footer saves through the calendar
API (currently a `TODO` — `/ice-calls/calendar` is GET-only). No drawer hand-off.

## Do not

- Set `disabled` because the screen is named "details", "preview", "summary", or "view"
- Set `disabled` because the API to save it does not exist yet — wire the submit to a `TODO`
- Read a gray box in your **own screenshot** as confirmation; compare against Figma's tokens
- Ship a control whose label renders as placeholder text when Figma floats it on the border
- Point an "Edit" button at a drawer that writes a different entity/endpoint
