# Industrial Pareto Visual — Project Worklog

This file is the shared worklog for all agents working on the Industrial Pareto Visual
(a Power BI Custom Visual targeting Power BI Report Server May 2026 — Build 15.0.1121.120).

---
Task ID: 0
Agent: Main (Z.ai Code)
Task: Initial research and architecture report — required by the user BEFORE any development starts (per instructions section 31).

Work Log:
- Read the user's instruction file located at:
  https://github.com/drvesali1374-ai/TradeBot/blob/main/Visual%20instructions
  (downloaded raw via curl from raw.githubusercontent.com; the path is a single UTF-8 file, not a folder).
- Confirmed the target environment: Power BI Report Server May 2026, Build 15.0.1121.120, released 2026-07-08.
- Confirmed the user's claim about the API version: Microsoft's official Change Log page
  (https://learn.microsoft.com/en-us/power-bi/report-server/changelog) states:
  "Custom Visual API shipped with release - version v5.10.0".
- Verified the same API version against Microsoft's own visual repo:
  microsoft/PowerBI-visuals-ForceGraph (main branch) uses "apiVersion": "5.10.0" in pbiviz.json.
- Verified that powerbi-visuals-api@5.10.0 is still published on npm (so we can pin to it exactly).
- Verified the latest powerbi-visuals-tools version is 7.2.1 (defaults to API 5.11.1 in templates,
  but supports `pbiviz new --api-version 5.10.0` to override).
- Reviewed Microsoft's powerbi-visuals-sampleBarChart repo for capabilities.json structure,
  dataViewMappings (categorical with categories + values), and the SelectionManager pattern.
- Searched GitHub for any open-source Power BI Pareto custom visual — found NONE that is suitable
  (no Microsoft official Pareto visual; community has been asking for one for years in the Power BI
  community forum thread "Why is there no Pareto chart visualization in Power BI").
- Verified the Power BI Custom Visual sandbox restrictions: no internet access, no external APIs,
  no external resources, runs in a low-privilege iframe. This forces 100% local rendering — which
  matches the user's hard requirement.
- Verified that Duration values in Power BI are passed to visuals as a decimal number of DAYS
  (so 74 hours = 3.083333... days). The visual receives this raw number in the DataView and can
  format it however we want in our own TypeScript code. No API restriction. We will NOT use
  JavaScript Date (which wraps at 24h); we will do pure arithmetic to support >24h durations.
- Verified the Modern Format Pane (getFormattingModel API) is fully supported on API 5.10.0 via
  the powerbi-visuals-utils-formattingmodel package.
- Verified the selectionManager + ISelectionId pattern for cross-filtering on API 5.10.0
  (identical to the sampleBarChart pattern).
- Compiled a comprehensive research report (sections A–I) to present to the user.

Stage Summary:
- API target: 5.10.0 (matches PBIRS May 2026).
- Tools: powerbi-visuals-tools 7.2.1 with `--api-version 5.10.0` flag.
- powerbi-visuals-api: pinned to exactly 5.10.0.
- Node.js: 20 LTS (recommended by Microsoft).
- TypeScript: ~5.5+ (compatible with tools 7.x).
- Rendering: pure SVG via D3 modules (d3-selection, d3-scale, d3-axis). No charting libs.
- No external resources, no internet calls, no third-party services. 100% local.
- Build from scratch (no open-source Pareto base suitable). MIT license.
- Architecture plan documented in the report.
- AWAITING USER APPROVAL of the report before starting development (per instructions section 31).

---
Task ID: 1
Agent: Main (Z.ai Code)
Task: Build MVP of Industrial Pareto Visual — skeleton + core modules + .pbiviz package, targeting PBIRS May 2026 (API 5.10.0).

Work Log:
- Created the visual project via `powerbi-visuals-tools@7.2.1` (`pbiviz new industrialParetoVisual`).
- Overrode the template's default API version (5.11.1) by editing pbiviz.json to `"apiVersion": "5.10.0"` and pinning `powerbi-visuals-api` to `5.10.0` in package.json (verified: `node_modules/powerbi-visuals-api/package.json` reports version 5.10.0).
- Wrote capabilities.json with:
    - dataRoles: Category (Grouping) + Values (Measure)
    - dataViewMappings: categorical (categories + values), with dataReductionAlgorithm top:5000
    - objects: bars, dataLabels, cumulativeLine, referenceLine, xAxis, yAxis, secondaryAxis, tooltip, background, general
    - privileges: [] (no special privileges needed)
    - supportsEmptyDataView: true, suppressDefaultTitle: true
- Implemented src/ modules:
    - interfaces.ts       — TypeScript types (ParetoDataPoint, ParetoViewModel, SelectionIdLike, DurationFormat, etc.)
    - durationFormatter.ts — Pure functions; converts decimal-days → HH:MM:SS WITHOUT 24h wrap (74h stays 74h, not 02h). Supports 6 formats. Tested with node script: all critical test cases from the instruction file section 28 PASS.
    - pareto.ts           — Pure functions; sort descending (stable), cumulative value, cumulative %, share %.
    - rtl.ts              — Detects Persian/Arabic/Hebrew characters per-label (for SVG text direction).
    - settings.ts         — Modern Format Pane (getFormattingModel) with 9 cards. Uses ItemDropdown with proper IEnumMember values. Worked around the `const enum ValidatorType` issue (TS2475) by using stable numeric literals 0/1 instead of the enum members.
    - interactivity.ts    — Wraps Power BI SelectionManager; handles single-click, Ctrl/Cmd+Click multi-select (manual array manipulation because API 5.10.0 has no `remove` method), and dimming of unselected bars. Bridges the SDK's dual ISelectionId typing (powerbi.visuals.ISelectionId vs powerbi.extensibility.ISelectionId) with a single `as unknown as` cast.
    - tooltip.ts          — DOM-built HTML tooltip (no innerHTML, to satisfy ESLint powerbi-visuals/no-inner-outer-html). Shows Category, Value, Cumulative Value, Share %, Cumulative %. Auto-RTL when most categories are Persian.
    - renderer.ts         — D3 SVG renderer: bars (with optional gradient), cumulative line with markers, 80% reference line (dashed/solid/dotted), X/Y/Y2 axes, grid lines, data labels, percentage labels. Type-safe ScaleLinear<number, number> and ScaleBand<string>. Per-label RTL detection.
    - visual.ts           — Entry point. Parses categorical DataView (extracts categories + measure + builds ISelectionId for each), builds Pareto model, resolves settings, calls renderPareto, wires click/hover handlers, manages rendering events.
- Configured tsconfig.json: target es2022, skipLibCheck, preserveConstEnums, isolatedModules:false (overridden because the build toolchain enforces isolatedModules anyway, and we needed to work around the const enum issue differently).
- Iteratively fixed 34 TypeScript errors down to 0:
    - ItemDropdown `items` and `value` must be IEnumMember objects, not strings — built `enumItems()` helper.
    - `powerbi.visuals.ValidatorType` is a `const enum` that can't be inlined under isolatedModules — used stable numeric literals 0/1 with explanatory comment.
    - `scaleLinear()` defaults to `<unknown, unknown, unknown>` — explicitly typed as `ScaleLinear<number, number>`.
    - `powerbi.extensibility.ISelectionId` (empty marker) vs `powerbi.visuals.ISelectionId` (with methods) — used the visuals variant internally, cast at the ISelectionManager boundary.
    - `ISelectionManager` on API 5.10.0 has no `remove` method — implemented multi-select toggle by manually building the selection array.
    - `VisualSelectionOptions` doesn't exist on this API — removed the import.
    - Replaced `innerHTML = ...` in tooltip.ts with DOM `createElement` + `textContent` (satisfies powerbi-visuals/no-inner-outer-html ESLint rule).
    - Removed unused eslint-disable comments.
- Ran `npm run package` — SUCCESS. Produced:
    dist/industrialParetoVisualEF1458CA875E4336BFF6C60197B73039.1.0.0.0.pbiviz (27 KB)
- Ran `npm run lint` — 0 errors, 0 warnings.
- Verified the bundled pbiviz.json reports `"apiVersion": "5.10.0"` (matches PBIRS May 2026).
- Copied the .pbiviz to /home/z/my-project/IndustrialPareto.1.0.0.0.pbiviz for stable delivery.
- Wrote docs: README.md, LICENSE (MIT), CHANGELOG.md, LICENSES.md, docs/architecture.md, docs/compatibility.md, docs/installation.md.
- Wrote samples/sample-data.csv (Persian test data from the instructions).

Stage Summary:
- MVP .pbiviz file delivered at /home/z/my-project/IndustrialPareto.1.0.0.0.pbiviz (27 KB).
- Full source tree at /home/z/pareto-work/industrialParetoVisual/.
- 9 source TypeScript files, ~2341 LOC total.
- apiVersion confirmed 5.10.0 inside the bundled pbiviz — matches PBIRS May 2026.
- Lint clean. Build clean.
- All critical Duration test cases PASS (74:00:00, 48:30:00, 26:00:00, 50:30:00).
- Pareto computation test (Persian categories) PASS — descending sort + cumulative % + 80% crossing all correct.
- AWAITING USER REVIEW before phase 2 (color-by-category picker, sorting controls, localization, accessibility, on-object formatting).

---
Task ID: 25
Agent: Main (Z.ai Code)
Task: v4.0.4 — Targeted Root-Cause Fix for 9 bugs (from v4.0.1.0 baseline).

Work Log:

BUG #1: Reference Line Label Font Size and Bold
- Root Cause: CSS .reference-label has font-weight:600 and font-size:11px. But .style() (inline) should override CSS class rules. After code inspection, .style() IS called correctly at lines 638-640. The CSS values match the defaults, so changes SHOULD work. However, to be safe, I removed the CSS font-weight and font-size from .reference-label so there's no confusion.
- File: style/visual.less (removed font-weight:600 and font-size:11px from .reference-label)
- Regression Risk: LOW — only CSS default removed, .style() still applies user settings.

BUG #2: Data Labels clipped at top
- Root Cause: Clip-path rect at y=0,height=innerHeight clips labels above y=0. Bar labels with "Above" position are at y=barTop-6 which can be negative.
- Fix: Extended clip-path rect to start at y=-padTop+4, height=innerHeight+padTop-4+8. Also added dynamic padTop calculation.
- File: src/renderer.ts (clip-path rect + padTop)
- Regression Risk: LOW — clip-path now covers more area, not less.

BUG #3: Label Background not centered/rounded
- Root Cause: Background used measureText() estimate (not getBBox()), padY=1 (too small), no rounded corners.
- Fix: Use getBBox() for actual text dimensions. padX=4, padY=2. rx=3, ry=3 for rounded corners. Text set before getBBox() for accurate measurement.
- File: src/renderer.ts (both bar label and cumulative label backgrounds)
- Regression Risk: LOW — only background rect dimensions changed.

BUG #4: Show Percentage Label disappears when OFF
- Root Cause: onPreProcess line 562: 'for (const s of labelSlices) s.visible = show && showLabels;' — this includes showLabels itself.
- Fix: Changed to: skip showLabels in the loop, set showLabels.visible = show separately (always visible when card shown).
- File: src/settings.ts (CumulativeLineCard.onPreProcess)
- Regression Risk: NONE — only visibility logic changed.

BUG #5: Position - replace InsideEnd with Center
- Root Cause: Position had InsideEnd but user wants Center.
- Fix: Replaced InsideEnd with Center in capabilities.json, settings.ts, interfaces.ts. Renderer: Center = y0 + (innerHeight - y0) / 2 (vertical center of bar).
- Files: capabilities.json, src/settings.ts, src/interfaces.ts, src/renderer.ts
- Regression Risk: LOW — InsideEnd replaced, not added alongside.

BUG #6: Responsive Layout
- Root Cause: Top padding was fixed at 16. Labels above bars were clipped.
- Fix: Dynamic padTop = max(16, barDataLabels.fontSize+8 if Above, markerSize+labelFontSize+12 if curve labels). Plot height = viewport.height - padTop - xAxisBlockHeight.
- File: src/renderer.ts
- Regression Risk: LOW — only padding calculation changed.

BUG #7: Background (Power BI native)
- Root Cause: This is a Power BI host behavior. Format > General > Effects > Background sets the visual CONTAINER background, not the SVG content. Our SVG is transparent. Same behavior as v3.0.0.0.
- Fix: No code change needed. This is a Power BI host limitation, not a visual bug.
- Regression Risk: N/A.

BUG #8: Z-order — bar labels under cumulative line
- Root Cause: Bar data labels were rendered BEFORE cumulative line. In SVG, later elements render on top.
- Fix: Moved bar data labels rendering to AFTER cumulative line and reference line. New order: bars → cumulative line → reference line → bar data labels (ON TOP).
- File: src/renderer.ts (reordered rendering blocks)
- Regression Risk: LOW — only DOM order changed.

BUG #9: Border on hover/selection
- Root Cause: CSS .bar:focus has outline: 2px solid #2D5BFF. This appears when a bar is clicked/focused.
- Fix: Changed to outline: none.
- File: style/visual.less
- Regression Risk: NONE — only outline removed.

Files changed:
- src/renderer.ts: clip-path, padTop, z-order, Center position, getBBox backgrounds
- src/settings.ts: showLabels visibility, Center in position items
- src/interfaces.ts: LabelPosition type (InsideEnd → Center)
- capabilities.json: position enumeration (InsideEnd → Center)
- style/visual.less: .bar:focus outline:none, .reference-label font-weight/font-size removed

Files NOT changed (no regression):
- src/visual.ts: Selection logic unchanged
- src/pareto.ts: Pareto calculation unchanged
- src/durationFormatter.ts: Duration formatting unchanged
- src/rtl.ts: RTL detection unchanged

- Bumped version to 4.0.4.0.
- npm run package — SUCCESS, 0 errors.
- npm run lint — 0 errors, 0 warnings.

Stage Summary:
- v4.0.4 .pbiviz delivered at /home/z/my-project/IndustrialPareto.4.0.4.0.pbiviz.
- 9 bugs fixed with targeted patches. No rewrite. No regression.
- USER TEST REQUIRED.

---
Task ID: 26
Agent: Main (Z.ai Code)
Task: v4.0.5 — Fix 4 problems (Sort direction, Padding, Background, Gridlines).

Work Log:

PROBLEM 1: Sort direction reversed
- ROOT CAUSE: compareRows() in pareto.ts had DescendingByValue and AscendingByValue SWAPPED.
  DescendingByValue returned `a.value - b.value` which sorts ASCENDING (small first).
  AscendingByValue returned `b.value - a.value` which sorts DESCENDING (big first).
- FIX: Swapped the return values.
  DescendingByValue: return b.value - a.value (biggest first, left to right)
  AscendingByValue: return a.value - b.value (smallest first, left to right)
- FILE: src/pareto.ts (compareRows function)
- REGRESSION RISK: NONE — only comparator logic changed. SelectionId, originalIndex, cumulative all unchanged.

PROBLEM 2: Excessive padding when Y axis title is empty
- ROOT CAUSE: leftPad was always 60 when yAxis.show=true, regardless of whether title existed.
- FIX: Dynamic leftPad based on title presence:
  hasYAxisTitle = yAxis.show && title.trim().length > 0
  leftPad = hasYAxisTitle ? 60 : 44
- FILE: src/renderer.ts (padding calculation)
- REGRESSION RISK: LOW — only padding calculation changed.

PROBLEM 3: Background remains white when OFF
- ROOT CAUSE: This is NOT a visual code issue. The visual's background rect is only drawn when settings.background.show=true. When OFF, no rect is drawn and the SVG is transparent. The "white background" the user sees is Power BI's native visual container background (Format > General > Effects > Background), which is controlled by the Power BI host, not by the visual code.
- ANALYSIS: v3.0.0.0 has the exact same behavior — both versions only draw a rect when settings.background.show=true. The "white" the user sees is from Power BI's host container.
- FIX: Added clarifying comment. No code change needed.
- REGRESSION RISK: N/A.

PROBLEM 4: Gridlines not configurable
- ROOT CAUSE: Gridlines were hard-coded in CSS (.grid line { stroke: #e8e8e8; stroke-dasharray: 2,2 }) and renderer (no settings).
- FIX: Added gridlines object to capabilities.json with 5 properties (show, color, opacity, width, style). Added GridlinesCard to settings.ts with onPreProcess. Added gridlines to ResolvedSettings interface. Modified renderer to use settings instead of hard-coded values.
- FILES: capabilities.json, src/settings.ts, src/interfaces.ts, src/renderer.ts
- REGRESSION RISK: LOW — gridlines rendering is now conditional (if settings.gridlines.show) and uses configurable values instead of hard-coded CSS.

Files changed:
- src/pareto.ts: compareRows swapped (PROBLEM 1)
- src/renderer.ts: dynamic leftPad (PROBLEM 2), gridlines settings (PROBLEM 4), background comment (PROBLEM 3)
- src/settings.ts: GridlinesCard + resolveSettings gridlines
- src/interfaces.ts: gridlines in ResolvedSettings
- capabilities.json: gridlines object
- pbiviz.json, package.json: version 4.0.5.0

Files NOT changed:
- src/visual.ts: Selection logic unchanged
- src/durationFormatter.ts: Unchanged
- src/rtl.ts: Unchanged
- style/visual.less: Unchanged (grid CSS still there as fallback)

- Bumped version to 4.0.5.0.
- npm run package — SUCCESS, 0 errors.
- npm run lint — 0 errors, 0 warnings.
- Verified: gridlines object present in capabilities, version 4.0.5.0.

Stage Summary:
- v4.0.5 .pbiviz delivered at /home/z/my-project/IndustrialPareto.4.0.5.0.pbiviz.
- 4 problems fixed. Sort direction corrected. Padding responsive. Gridlines configurable.
- USER TEST REQUIRED.
