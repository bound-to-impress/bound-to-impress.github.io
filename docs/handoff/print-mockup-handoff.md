# Handoff: Print Mockup Feature — Guillotine Cut Planner

## Status at end of session

`guillotine-cut-planner.html` is fully implemented and tested. All acceptance scenarios pass including gutters, landscape sheets, no-cuts case, unit conversion, presets, and orientation toggles.

A **Print Mockup** feature has been wired in (button appears, `window.print()` fires, print CSS swaps visibility) but the **visual rendering approach needs to be reconsidered** before the next session continues.

---

## What is already in the file

In `guillotine-cut-planner.html`:

- `<style>` block contains `@media print` and `@page` rules that hide `body > main` and show `#mockupPage` when printing.
- `#mockupPage` div exists after `</main>` — contains title, subtitle span, `#mockupSvg` div, and safety footer.
- **Print Mockup** button at the bottom of `#results` — calls `generateMockupSVG(currentPlan)`, inserts the result into `#mockupSvg`, updates `#mockupSubtitle`, then calls `window.print()`.
- `generateMockupSVG()` is implemented as an SVG-based renderer (see critique below — this may be replaced).
- `currentPlan` is set on every successful form submission and cleared on `clearResults()`.
- `planCuts()` now returns `sheetWmm`, `sheetHmm`, `rows`, `cols` in addition to the previous fields.

---

## Open decision: SVG vs HTML/CSS div approach

The user questioned whether SVG is necessary at all, noting that plain HTML/CSS `<div>` elements with borders could achieve the same result more simply. Example shared by user:

```html
/* A div with a full border outline */
.top-box { border: 2px solid #000; width: 300px; height: 150px; }

/* A div with only a left border */
.bottom-box { border-left: 1px solid #000; width: 300px; height: 150px; }
```

### Assessment

**HTML/CSS divs are viable and arguably better for this use case:**

- The layout is a regular grid — CSS `display: flex` or `display: grid` with fixed `width`/`height` in mm units handles this naturally.
- CSS `mm` units (`width: 100mm`) are respected by browser print engines and map directly to physical millimetres on the printed page.
- Borders handle the piece outlines cleanly. Crop marks can be achieved with pseudo-elements (`::before` / `::after`) on corner divs — no SVG maths required.
- The entire layout reflows correctly when the browser scales to fit the paper.
- Much less code, no viewBox arithmetic, no n3() rounding helpers needed.

**Where SVG has a genuine advantage:**
- Crop marks that extend *beyond* the sheet boundary (into the margin) are easier in SVG since SVG overflow is visible by default.
- The clip-path trick used to suppress interior crop marks on zero-gutter layouts is clean in SVG.

**Recommendation for next session:** Replace `generateMockupSVG()` with a `generateMockupHTML()` function that builds a div-based layout. Use CSS `mm` units for all dimensions. Handle crop marks with absolutely-positioned pseudo-elements or small `<div>` elements at each piece corner.

---

## Second piece of feedback: centre the production block on the mockup

The user noted that pre-press operators typically centre the work area on the page. The current SVG places the production block at origin (top-left), with waste shown to the right and bottom.

**Required change:** The printed mockup should centre the full sheet on the print page, and within the sheet the production block should ideally be centred (or at least clearly shown with the waste areas labelled on all sides where they exist).

For the div approach this is easy: wrap the sheet div in a flex container with `justify-content: center; align-items: center; min-height: 100vh;` in the print CSS. The sheet div itself (physical mm dimensions) sits centred on the page.

---

## Mockup visual spec (from PDF reference provided by user)

A sample print job PDF was reviewed: `a4-crops-and-art-outline.pdf` (3×3 grid of MTG cards on A4). Key observations:

- **Crop marks** appear at all 4 corners of each individual piece — L-shaped, two short perpendicular lines per corner.
- Marks start with a small gap from the piece edge (~1–2mm) and extend outward ~5mm.
- In the **gutter space** between pieces, interior crop marks from adjacent pieces face each other and fit within the gutter — they do NOT overlap.
- On the **outer edges** of the grid, crop marks extend into the bleed/margin area beyond the sheet or beyond the sheet boundary.
- The **piece outlines** are the actual card artwork (we use light grey fill `#d8d8d8` as a stand-in).
- The **sheet boundary** has a thin line.
- No cut lines drawn between pieces — only the crop marks indicate where cuts happen.
- White gutter space clearly separates pieces.

---

## Suggested div-based implementation outline

```javascript
function generateMockupHTML(plan) {
    const { sheetWmm, sheetHmm, reqW, reqH,
            finWmm, finHmm, rows, cols,
            gutterMm, excessW, excessH } = plan;

    const pitchW = finWmm + gutterMm;
    const pitchH = finHmm + gutterMm;

    // Crop mark size in mm
    const cmLen = 5, cmGap = 1.5, cmThick = 0.3;

    // Sheet wrapper (physical mm size)
    // Production block offset from sheet origin = (0, 0) top-left
    // Waste shown right and bottom via the sheet being larger than reqW/reqH

    let pieces = '';
    for (let r = 0; r < rows; r++) {
        for (let c = 0; c < cols; c++) {
            const left = c * pitchW;
            const top  = r * pitchH;
            // Piece div + 4 corner crop mark divs
            pieces += `
                <div style="position:absolute;
                            left:${left}mm; top:${top}mm;
                            width:${finWmm}mm; height:${finHmm}mm;
                            background:#d8d8d8;
                            box-sizing:border-box;
                            border:0.2mm solid #888;">
                    <!-- crop mark divs at each corner go here -->
                </div>`;
        }
    }

    // Waste area overlays (right strip, bottom strip)
    const wasteRight  = excessW > 0 ? `<div style="position:absolute;left:${reqW}mm;top:0;width:${excessW}mm;height:${sheetHmm}mm;background:#f0f0f0;"></div>` : '';
    const wasteBottom = excessH > 0 ? `<div style="position:absolute;left:0;top:${reqH}mm;width:${sheetWmm}mm;height:${excessH}mm;background:#f0f0f0;"></div>` : '';

    return `
        <div style="position:relative;
                    width:${sheetWmm}mm; height:${sheetHmm}mm;
                    background:white;
                    border:0.3mm solid #000;
                    box-sizing:border-box;">
            ${wasteRight}
            ${wasteBottom}
            ${pieces}
        </div>`;
}
```

Crop mark divs at each corner can be small absolutely-positioned `<div>` elements using `border-top`/`border-left` etc., or `::before`/`::after` pseudo-elements. The crop mark div is positioned slightly outside the piece edge using negative `top`/`left` values.

---

## Files to be aware of

| File | Role |
|------|------|
| `guillotine-cut-planner.html` | Main implementation — fully working except mockup renderer needs replacement |
| `docs/requirements/guillotine-cut-planner.md` | Canonical requirements |
| `docs/prompts/guillotine-cut-planner.md` | Implementation prompt |
| `/mnt/d/Downloads/a4-crops-and-art-outline.pdf` | Reference PDF showing expected crop mark style (3×3 card imposition) |

---

## Next session tasks

1. **Replace `generateMockupSVG()`** with a div/CSS-based `generateMockupHTML()` function inside `#mockupPage`'s `#mockupSvg` container.
2. **Centre the mockup** on the print page using flex centering in `@media print` CSS.
3. **Implement crop marks** as positioned `<div>` elements at each piece corner, using CSS borders for the L-shape.
4. **Test print output** across the key scenarios: no gutter (4×3), with gutter (4×3 + 5mm), landscape, single piece exact fit.
5. Remove the now-unused `generateMockupSVG()` and `n3()` functions from the script.
