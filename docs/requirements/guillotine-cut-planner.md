# Guillotine Cut Planner Requirements

## Purpose

Provide a browser-based planner that converts a rectangular sheet layout into an ordered set of guillotine instructions. The planner is an aid for calculating measurements; it does not replace operator training, machine procedures, or independent measurement checks.

## Units

The planner supports millimetres and inches.

- A unit selector (mm / in) is visible at the top of the form.
- All dimension inputs and all displayed measurements use the active unit.
- When the unit changes, every numeric value currently in the form is converted and the fields are updated in place. Gutter, sheet dimensions, and finished dimensions are all converted.
- Conversion factor: 1 inch = 25.4 mm exactly.
- Converted values are rounded to two decimal places when displayed in inches and to two decimal places when displayed in millimetres. Rounding is for display and input only; internal calculations use the unrounded converted value.
- Preset values (see below) are stored in millimetres and converted for display in the active unit.
- The selected unit is persisted in `localStorage` under the key `guillotineUnit` and restored on page load.

## Inputs

All dimensions are entered in the active unit.

- Sheet size: width and height.
- Finished piece size: width and height.
- Rows: the number of finished-piece heights laid out across the sheet height.
- Columns: the number of finished-piece widths laid out across the sheet width.
- Gutter: the uniform space between adjacent pieces in the layout (see below).

Dimensions must be positive finite numbers. Rows and columns must be positive whole numbers. The planner does not automatically rotate the finished piece or interchange rows and columns.

### Gutter

The gutter is the gap between adjacent pieces within the layout grid. It is used to accommodate print bleed or a safe margin between cuts.

- The gutter applies between pieces only, not around the outer perimeter of the production block.
- The gutter must be zero or a positive number strictly less than 15 mm (or its equivalent in the active unit: less than 0.5906 in).
- Default value is 0.
- An input mode of `decimal` and a step of `any` apply.
- When the gutter is 0, no gutter-strip discard cuts are emitted.

### Sheet size presets

A preset selector beside the sheet size fieldset populates the width and height fields with standard sheet dimensions. Preset values in millimetres:

| Label | Width (mm) | Height (mm) |
|-------|-----------|------------|
| SRA3  | 320       | 450        |
| A3    | 297       | 420        |
| A4    | 210       | 297        |

Selecting a preset overwrites the width and height fields. The selector reverts to a neutral "Select preset…" state when either field is edited manually.

### Finished size presets

A preset selector beside the finished size fieldset populates the finished width and height fields. Preset values in millimetres:

| Label | Width (mm) | Height (mm) |
|-------|-----------|------------|
| A4    | 210       | 297        |
| A5    | 148       | 210        |
| A6    | 105       | 148        |
| Card  | 63        | 88         |

Behaviour matches the sheet size preset selector.

### Print orientation toggles

Each size fieldset (sheet size and finished size) includes a portrait/landscape orientation toggle.

- The toggle displays an icon indicating the current orientation (portrait when height ≥ width, landscape when width > height).
- Activating the toggle swaps the width and height values within that fieldset only.
- The toggle does not affect any other fields and does not interact with the preset selector (after swapping, the preset selector reverts to its neutral state).
- If the fields are empty the toggle has no effect.

## Layout and validation

The required production-block dimensions are:

- Required width: `columns × finishedWidth + (columns − 1) × gutter`
- Required height: `rows × finishedHeight + (rows − 1) × gutter`
- Finished quantity: `rows × columns`

The layout is valid only when the required width does not exceed the sheet width and the required height does not exceed the sheet height. An invalid layout must not generate cutting commands. Its error must identify the failing axis and state how many millimetres (or inches) additional sheet size would be needed on that axis.

Excess stock is removed from one outside edge on each axis. The production block remains flush to the backgauge datum; excess is not centred. Blade kerf is treated as negligible.

## Cutting sequence

### Initial orientation

The first instruction must tell the operator to place the sheet's shortest edge against the backgauge.

- When sheet width is shorter than sheet height, place the width edge against the backgauge and process rows (the height axis) first.
- When sheet height is shorter than sheet width, place the height edge against the backgauge and process columns (the width axis) first.
- For a square sheet, treat the entered width edge as the backgauge edge and process rows first.

### Processing an axis

For an axis with sheet dimension `S`, finished dimension `F`, piece count `N`, and gutter `G`:

1. Calculate `requiredDimension = N × F + (N − 1) × G`.
2. **Trim step** (omit when `S = requiredDimension`): set the backgauge to `requiredDimension`, cut off the excess at the front, discard that offcut, and retain the required block against the backgauge.
3. **Subdivision** (omit entirely when `N = 1`): for `k` from `N − 1` down to `1`:
   a. **Piece cut**: set the backgauge to `k × (F + G)`. Cut. Retain the strip of depth `F` separated at the front. Retain the block of depth `k × (F + G)` remaining against the backgauge.
   b. **Gutter cut** (omit when `G = 0`): set the backgauge to `k × F + (k − 1) × G`. Cut. Discard the gutter strip of depth `G` at the front. Retain the block remaining against the backgauge.
4. After the final iteration, the remaining block equals `F` and is the first piece; retain it.
5. At the last piece cut (`k = 1`), the step 3a cut produces the second piece and the step 3b cut (if `G > 0`) discards the gutter between the first and second pieces. The remaining block is then the first piece.

**Special case — single piece, exact fit**: when `N = 1` and `S = F`, there is no trim and no subdivision. Emit the initial orientation instruction and report "No cuts required."

### Between axes

After completing the first axis, instruct the operator to:

1. Align all equal strips and stack them.
2. Rotate the stack 90 degrees.
3. Place the edge whose length equals the first-axis finished dimension against the backgauge.
4. Note that the stack must remain within the machine's documented capacity; otherwise the operator must repeat the second-axis sequence for each safe-sized batch.

The second axis is then processed using the same trim-and-subdivide procedure.

**Suppress the between-axes block** when both axes required no cuts (trim and subdivision both omitted on both axes). Emit only the orientation instruction and "No cuts required" in that case.

## Generated result

After successful submission, show:

- The production-block dimensions (required width × required height).
- The expected finished quantity and finished-piece dimensions.
- Excess on the width and height axes. If excess is zero on an axis, state "No excess" for that axis.
- A numbered, ordered command list with clear stage headings.
- For every command: orientation where relevant, exact backgauge setting, the cut action, and what to retain or discard.

All measurements are displayed in the active unit using no unnecessary trailing zeroes. Backgauge settings in the command list use tabular numerals.

The result must include a prominent warning to verify every measurement before cutting and to use the commands only when trained and while following the guillotine manufacturer's procedures.

## Interface and accessibility

- The page title and heading are **Guillotine Cut Planner**.
- A unit selector (mm / in) appears at the top of the form.
- The form groups inputs into clearly labelled fieldsets: Sheet Size, Finished Size, and Layout.
- Each size fieldset includes a preset selector and an orientation toggle.
- The Layout fieldset contains Rows, Columns, and Gutter.
- A **Generate Commands** submit button triggers validation and generation.
- Commands remain hidden until a valid submission succeeds.
- Validation errors appear beside or immediately below the form control or group they relate to and are announced accessibly.
- Generated results use an `aria-live="polite"` region and tabular numerals.
- Every input has a programmatically associated label and an appropriate decimal or numeric input mode.
- The layout remains usable without horizontal scrolling at mobile widths and supports keyboard operation.
- Light and dark themes follow the existing calculators, including the persisted `darkMode` preference.

## Technical constraints

- Implement the planner as `guillotine-cut-planner.html`.
- Keep application markup and JavaScript in that file.
- Use the repository-local `assets/vendor/tailwind/browser.js` and Material Icons stylesheet, consistent with `cubic-calculator.html` and `paper-weight-calculator.html`.
- Do not add a framework, package dependency, network dependency, or backend.
- Do not change `index.html` as part of this feature.
- Keep calculation and sequence generation separate from DOM rendering so their behaviour can be tested independently.
- Do not add gradients or decorative animation.

## Acceptance scenarios

### Portrait sheet with waste on both axes, no gutter

For a `450 × 640 mm` sheet, `100 × 150 mm` finished size, 4 rows, 3 columns, gutter 0 mm:

- Required width: 3 × 100 = 300 mm. Required height: 4 × 150 = 600 mm.
- Width excess: 150 mm. Height excess: 40 mm.
- Sheet width (450) < sheet height (640): place the 450 mm edge against the backgauge; process rows/height axis first.
- Height axis cuts: trim at 600 mm (discard 40 mm offcut); subdivide at 450 mm, 300 mm, 150 mm (no gutter cuts).
- Resulting strips: four strips of 450 × 150 mm.
- Stack strips, rotate, place 150 mm edge against backgauge.
- Width axis cuts: trim at 300 mm (discard 150 mm offcut); subdivide at 200 mm, 100 mm (no gutter cuts).
- Output: 12 pieces at 100 × 150 mm.

### Portrait sheet with waste on both axes and 3 mm gutter

For a `450 × 640 mm` sheet, `100 × 150 mm` finished size, 4 rows, 3 columns, gutter 3 mm:

- Required width: 3 × 100 + 2 × 3 = 306 mm. Required height: 4 × 150 + 3 × 3 = 609 mm.
- Width excess: 144 mm. Height excess: 31 mm.
- Place the 450 mm edge against the backgauge; process rows/height axis first.
- Height axis (F = 150, G = 3, pitch = 153):
  - Trim at 609 mm (discard 31 mm).
  - k = 3: piece cut at 459 mm, retain 150 mm strip (row 4); gutter cut at 456 mm, discard 3 mm.
  - k = 2: piece cut at 306 mm, retain 150 mm strip (row 3); gutter cut at 303 mm, discard 3 mm.
  - k = 1: piece cut at 153 mm, retain 150 mm strip (row 2); gutter cut at 150 mm, discard 3 mm.
  - Remaining block (150 mm) = row 1, retain.
- Stack four 450 × 150 mm strips, rotate, place 150 mm edge against backgauge.
- Width axis (F = 100, G = 3, pitch = 103):
  - Trim at 306 mm (discard 144 mm).
  - k = 2: piece cut at 206 mm, retain 100 mm strip (col 3); gutter cut at 203 mm, discard 3 mm.
  - k = 1: piece cut at 103 mm, retain 100 mm strip (col 2); gutter cut at 100 mm, discard 3 mm.
  - Remaining block (100 mm) = col 1, retain.
- Output: 12 pieces at 100 × 150 mm.

### Additional cases

- A landscape sheet (width > height) processes the width/columns axis first because the height edge is shortest.
- An exact-fit grid (no excess on either axis) omits both trim cuts but still emits subdivision cuts.
- One row omits subdivision cuts for the height axis; one column omits them for the width axis.
- A single finished piece exactly matching the sheet reports "No cuts required."
- A square sheet follows the width-edge and rows-first tie rule.
- Setting gutter to 0 produces identical output to the original no-gutter scenario.
- Missing values, zero or negative dimensions, non-finite dimensions, fractional row or column counts, a gutter of 15 mm or more, and oversized production blocks prevent generation and show actionable errors.
- Repeated calculations replace the previous command list without duplicating steps.
- Unit switching converts all field values; preset selection populates correctly in both units.
- Orientation toggles swap width and height values within the affected fieldset.
- The planner remains readable and operable on mobile and in both light and dark themes.
