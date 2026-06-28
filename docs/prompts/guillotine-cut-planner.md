# Build the Guillotine Cut Planner

Implement `guillotine-cut-planner.html` according to `docs/requirements/guillotine-cut-planner.md`. Treat the requirements document as the canonical source when this prompt is abbreviated or ambiguous.

## Goal

Build a responsive, repository-contained calculator that accepts a sheet size, finished size, rows, columns, and gutter, then generates safe-to-read, ordered guillotine measurements and handling instructions. The page must visually belong beside `cubic-calculator.html` and `paper-weight-calculator.html`.

## Page structure

- Create a single calculator card titled **Guillotine Cut Planner**.
- Use the existing slate light/dark palette, blue accent, rounded card, default Tailwind shadow, and compact responsive spacing.
- Load only `assets/vendor/tailwind/browser.js` and `assets/vendor/material-icons/material-icons.css`; keep all application JavaScript inline.
- Add an accessible icon-only dark-mode button. Initialise it from the `darkMode` localStorage key, falling back to `prefers-color-scheme`, and persist changes.

### Unit selector

- Render a segmented mm / in toggle at the top of the form, outside any fieldset.
- Persist the selected unit in `localStorage` under `guillotineUnit` and restore on page load.
- When the unit changes, convert every numeric field value currently in the form (sheet width, sheet height, finished width, finished height, gutter) by multiplying or dividing by 25.4 and rounding to two decimal places. Do not convert rows or columns.
- The conversion happens immediately on toggle change without requiring form resubmission.

### Fieldsets

Use semantic `<fieldset>` / `<legend>` for three groups:

**Sheet Size** — width and height, both `type="number"` with `min="0.01"`, `step="any"`, `inputmode="decimal"`. Include:
- A "Select preset…" `<select>` that, when changed to a named option, populates the width and height fields with the preset value converted to the active unit. Revert the select to its placeholder state whenever either field is edited manually. Presets (stored in mm): SRA3 320 × 450, A3 297 × 420, A4 210 × 297.
- A portrait/landscape orientation toggle button (use `portrait` and `crop_landscape` Material Icons). Clicking it swaps the current width and height values. If the fields are empty the button has no effect. After a swap, revert the preset select to its placeholder state.

**Finished Size** — finished width and height, same input attributes as sheet size. Include:
- A "Select preset…" `<select>` with the same behaviour. Presets (stored in mm): A4 210 × 297, A5 148 × 210, A6 105 × 148, Card 63 × 88.
- A portrait/landscape orientation toggle button with the same swap behaviour.

**Layout** — three inputs:
- Rows: `type="number"`, `min="1"`, `step="1"`, `inputmode="numeric"`.
- Columns: `type="number"`, `min="1"`, `step="1"`, `inputmode="numeric"`.
- Gutter: `type="number"`, `min="0"`, `max` set dynamically (14.9999 mm or 0.5905 in, update when unit changes), `step="any"`, `inputmode="decimal"`. Label as "Gutter (bleed/safe margin)". Default value 0.

Add a full-width **Generate Commands** submit button.

Place validation feedback near the form and render successful output beneath it in an `aria-live="polite"` region.

## Calculation model

Implement all calculation and command generation as plain deterministic functions with no DOM reads or writes. The functions must be independently testable.

### Parse and validate

- Sheet width, sheet height, finished width, finished height, and gutter must be positive (> 0) finite numbers. Exception: gutter may be 0.
- Gutter must be strictly less than 15 mm (convert threshold to active unit: < 0.5906 in). Reject and show an error if this constraint is violated.
- Rows and columns must be positive integers.
- All values must be present and valid before any calculation proceeds.

### Convert inputs to millimetres

Always work internally in millimetres. If the active unit is inches, multiply all dimension inputs by 25.4 before calculation. Apply no further rounding to internal values.

### Core calculation

```
requiredWidth  = columns × finishedWidth  + (columns − 1) × gutter
requiredHeight = rows    × finishedHeight + (rows    − 1) × gutter
quantity       = rows × columns
widthExcess    = sheetWidth  − requiredWidth
heightExcess   = sheetHeight − requiredHeight
```

Reject if `widthExcess < 0` or `heightExcess < 0`. Report the failing axis and the deficit.

### Axis-selection rule

- `sheetWidth < sheetHeight`: width edge to backgauge; process height axis (rows) first.
- `sheetHeight < sheetWidth`: height edge to backgauge; process width axis (columns) first.
- Equal dimensions: width edge to backgauge; process height axis (rows) first.

### Processing one axis

Function signature (conceptual): `processAxis(S, F, N, G)` → array of command objects.

```
requiredDimension = N × F + (N − 1) × G
```

1. If `S > requiredDimension`: emit trim command at `requiredDimension`.
2. If `N > 1`: for `k` from `N − 1` down to `1`:
   a. Emit piece-cut command at `k × (F + G)`. Label: retain the separated strip; retain remaining block.
   b. If `G > 0`: emit gutter-cut command at `k × F + (k − 1) × G`. Label: discard the gutter strip; retain remaining block.
3. If `N > 1`: emit a final "retain remaining block as the first piece" note (no additional cut).

### Between-axes instruction

After the first axis: emit stack, rotate, and reposition commands referencing the first-axis finished dimension as the new backgauge edge. Include the machine-capacity caveat.

Suppress all between-axes content and the second-axis section when the total required cuts on both axes is zero (single exact-size piece).

### No-cuts case

When `quantity = 1` and `widthExcess = 0` and `heightExcess = 0`: emit only the orientation instruction and an explicit "No cuts required" message.

### Formatting

Format all millimetre values for display: strip unnecessary trailing zeroes (e.g., 150 not 150.0, 153.5 not 153.50). When the active unit is inches, format to two decimal places. Apply `tabular-nums` to all measurement figures.

Do not apply rounding inside the calculation functions; format only at the rendering layer.

## Result presentation

Show a concise summary before the command list:

- Production block: required width × required height (in active unit).
- Output: `quantity` pieces at finished width × finished height.
- Sheet excess: width axis and height axis. Show "No excess" when an axis is exact.
- Gutter used: the gutter value (omit when 0).

Render commands as a numbered list with clear stage headings ("Stage 1 — Height axis (rows)", etc.). Use tabular numerals for all backgauge settings. Keep content readable on a narrow screen.

Always include a visible safety warning near the result: verify every measurement before cutting; only trained operators following the machine manufacturer's procedures should use the generated commands. Do not imply the page verifies machine capacity or operational safety.

## Validation and interaction

- Do not show stale commands after an invalid submission; clear the results region.
- Show actionable, field-specific errors and move focus to the first invalid field or an error summary.
- Re-submission replaces the previous result.
- Preserve native keyboard behaviour and allow pasted values.
- Associate every label and error with its input using `for`/`id` or `aria-describedby`. Use semantic HTML before ARIA.
- Use `text-balance` for headings, `text-pretty` for explanatory copy, `tabular-nums` for measurements.
- Update the gutter `max` attribute when the unit selector changes so native browser validation reflects the correct threshold.
- Do not add gradients, decorative animation, custom colour systems, dialogs, remote resources, or new dependencies.

## Verification

Manually exercise all acceptance scenarios in `docs/requirements/guillotine-cut-planner.md`, including:

- `450 × 640` sheet, finish `100 × 150`, 4 rows, 3 columns, gutter 0 mm. Confirm the exact stage order and gauge settings (trim 600, subdivide 450 → 300 → 150; trim 300, subdivide 200 → 100).
- Same inputs with gutter 3 mm. Confirm alternating piece and gutter cuts on both axes (height: trim 609, 459/456, 306/303, 153/150; width: trim 306, 206/203, 103/100).
- A landscape sheet (width > height) that processes the width/columns axis first.
- Exact-fit grid (no excess): confirm trim cuts absent, subdivision cuts present.
- One-row and one-column layouts: confirm subdivision absent on the relevant axis.
- A single exact-size piece: confirm "No cuts required".
- A square sheet: confirm width-edge and rows-first tie rule.
- Gutter at exactly 15 mm (in mm mode) is rejected; 14.99 mm is accepted.
- Unit switch from mm to in and back: confirm field values convert correctly and presets populate in the active unit.
- Orientation toggles: confirm width and height values swap within the correct fieldset only.
- Every invalid-input category and oversized grid on each axis.
- Repeated valid and invalid submissions.
- Keyboard operation, mobile layout, light mode, dark mode, and persistence after reload.

Before finishing, confirm the page works when served with the repository's existing static files, introduces no network requests, and does not modify `index.html`.
