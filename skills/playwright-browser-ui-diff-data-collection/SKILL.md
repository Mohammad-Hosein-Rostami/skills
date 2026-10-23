---
name: playwright-browser-ui-diff-data-collection
description: Collect rich “evidence JSON” from a headed Playwright browser session (boxes + computed styles + form values) so we can compare App UI against Figma screenshots and metadata.
---

# Playwright UI Diff Data Collection

Use this skill as the “forensics” layer before/after a screenshot diff.

## Core goals

- Capture **layout geometry**: bounding boxes (`x,y,w,h`) via `browser_snapshot` with `boxes: true`.
- Capture **visual style** for mismatch candidates: computed styles (colors, font metrics, radius, shadows).
- Capture **stateful UI values**: inputs/select values/checked states.
- Output a single JSON evidence object you can reference in the final diff report.

## Mandatory prerequisites

1. Portal page is already open in the browser MCP session.
2. Viewport is correct (default **1530×920**, DSF 1) to keep screenshot and geometry comparable.
3. Elements use `data-test` when possible (repo standard).

## Evidence capture workflow

### Step 1 — Snapshot with boxes

1. Run `browser_snapshot` with `boxes: true`.
2. From the snapshot, extract (or let the orchestrator extract) the refs for:
   - all elements with `data-test`
   - key headings/buttons by role/name (e.g. `Capture`, `Retake`, `Save`)
   - optionally: any `data-node-id` attributes (generated-by-figma code)

### Step 2 — Computed style extraction (targeted)

For each “target ref” from Step 1, collect:

- bounding box: `{ x, y, width, height }`
- typography: `{ fontFamily, fontSize, fontWeight, lineHeight, letterSpacing }`
- colors: `{ color, backgroundColor }`
- borders: `{ borderRadius, borderColor, borderWidth }`
- shadows: `{ boxShadow, filter }` (if any)

Implementation detail: computed styles are read with `getComputedStyle(el)`.

### Step 3 — Form/state extraction

If the page contains an edit form, collect:

- text input values
- select/autocomplete selected value text
- checkbox/radio checked

Prefer reading:

- `value` for `<input>`
- visible selected label text for custom select components

### Step 4 — Save JSON evidence

Write the evidence object to:

- `temp/agent-browser/ui-diff-evidence/{slug}.json`

Also return it to chat (summarized) as:

- top 10 elements by “likely mismatch” (e.g. boxes delta, missing nodes)
- any missing `data-test` nodes you expected from the Figma structure

## Data mapping hints (App ↔ Figma)

- If elements include `data-node-id`, map directly.
- Otherwise map by stable semantics:
  - role + accessible name (heading text, button label)
  - `data-test` IDs
  - exact i18n strings visible in English dev locale

## Output template (what the orchestrator should print)

```json
{
  "route": "...",
  "viewport": "1530x920",
  "targets": [
    {
      "key": "scan-business-save-next",
      "ref": "f12e331",
      "bbox": { "x": 0, "y": 0, "w": 0, "h": 0 },
      "computedStyle": { "color": "...", "fontSize": "...", "borderRadius": "..." },
      "value": "..."
    }
  ]
}
```
