# Power BI Report Development Playbook

Created: 2026-11-09  
Source: the Operational Scorecard implementation and troubleshooting conversation for this PBIP project.  
Audience: future AI coding assistants and developers creating or extending Power BI reports from source.

## Start Here Next Time

Read this guide before editing a similar Power BI project. Treat it as a record of proven practices and observed failures, not permission to copy this model or overwrite an existing report.

1. Inspect the actual PBIP, report, semantic model, pages, visuals, themes, relationships, and source queries.
2. Read current files again when Desktop or the user has saved changes. Never overwrite newer edits using an earlier chat snapshot.
3. Verify source metadata and sample values rather than trusting expected field names or inferred data types.
4. State a short file-level implementation plan, identify a focused validation check, then make a small change.
5. Validate immediately. Distinguish JSON parsing, schema validation, TMDL deserialization, DAX evaluation, and visual rendering.
6. Preserve connections, relationships, existing identifiers, bookmarks, and interactions unless the requested behavior requires a change.
7. Reopen in Desktop and test real interactions before declaring the report functional.

For a future task, explicitly ask the assistant to read `Sample/POWER_BI_REPORT_PLAYBOOK.md`. This is a normal reference document, not an automatically loaded VS Code instruction file.

## Project Reference

This project's layout is:

```text
Score Card Prototype.pbip
Score Card Prototype.Report/
  definition.pbir
  definition/
    report.json
    version.json
    pages/
      pages.json
      <page-id>/
        page.json
        visuals/<visual-id>/visual.json
    bookmarks/
      bookmarks.json
      <bookmark-id>.bookmark.json
Score Card Prototype.SemanticModel/
  definition.pbism
  definition/
    database.tmdl
    model.tmdl
    relationships.tmdl
    tables/*.tmdl
Sample/
  Score card template.png
```

- The report connects to its semantic model through the existing relative path in `definition.pbir`.
- The main business tables are `OrgUnits`, `Calendar`, `Metrics`, `Goals`, and `Actuals`.
- Imported partitions reference an existing Excel workbook. Preserve those source expressions and paths; do not recreate the import.
- `Actuals` has district/metric/week observations. `Calendar` is weekly, not a continuous daily date table.
- Four business relationships connect OrgUnits, Calendar, and Metrics to Actuals, and Metrics to Goals. Two additional generated date relationships were retained.
- The actual Goals-to-Metrics relationship was one-to-one and bidirectional, not the prompt's expected one-to-many relationship. It worked and was preserved.
- The existing theme is Fluent 2. Theme defaults materially affect padding, borders, hover colors, and fonts.
- The original empty page was reused as `Scorecard` on a 1920 x 1080 canvas. Existing page and model identifiers were preserved.

These are reference facts for this project only. Rediscover the equivalent facts in every new project.

## Implementation Progress and Confidence

### Confirmed Through Desktop Feedback

- The project opens after the encoding, visual-nesting, and measure-expression fixes described below.
- Core scorecard measures evaluate after the reported DAX variable and indentation problems were repaired.
- Four metric-specific KPI arrangements display current values, goals, and weekly comparisons.
- The metric matrix displays YTD, goals, and four period-week results.
- The help button and open/close overlay worked after the user tested the interaction.
- Explicit button hover/pressed styling addressed the blue-on-blue problem.
- Disabling matrix-to-KPI cross-filtering stopped row selections from blanking unrelated KPI cards.
- The matrix has shown expanded metric detail in Desktop; retain the row-visibility protection when changing its layout.

### Latest Refinements: Source-Validated, Not Yet Desktop-Confirmed

The final request implemented these changes, but the conversation did not include a subsequent Desktop screenshot or runtime confirmation:

- White canvas, left-aligned text, simplified slicer containers, and an unframed matrix.
- Explicit matrix widths, a `Measure` header, group/detail shading, and a visible P9 default.
- Native `multiRowCard` visuals introduced to obtain left-aligned measure text after legacy card alignment settings were discarded by Desktop.
- Separate gray week/value captions and colored change captions for each KPI.
- Nine new formatting/text measures, including independent YTD and period-week status colors.
- Additional KPI text elements placed below the help overlay in stacking order.

Do not treat these last refinements as pixel-perfect or DAX-runtime-verified. Check them in Desktop before reusing them as a proven visual template.

## Inspect Before Implementing

Build a concise inventory before making changes:

- PBIP artifact registration and the report-to-model path.
- Actual schema versions declared by the project files. Use version-compatible definitions rather than upgrading versions unnecessarily.
- Existing pages, page order, active page, useful visuals, visual IDs, and parent groups.
- Table/column names, types, existing measures, sort settings, hidden fields, and source partitions.
- Actual relationship cardinalities, directions, and generated date-table relationships.
- Bookmarks, interaction overrides, theme defaults, and any existing local validation tooling.
- A reference image, if present, viewed directly rather than inferred from a text description.

This implementation found that `Metrics[NumberFormat]` contained strings such as `0.0%`, `0.0`, and `0`, although the original model and Power Query step typed it as numeric. Both the TMDL column type and the Power Query conversion needed correction to text. The workbook connection itself did not change.

## Model and Measure Rules

### Aggregation and Formats

- Avoid totals across unrelated metrics. Return blank when there is no single metric in context.
- Average percentage, rate, score, and productivity observations where appropriate; sum additive counts.
- This prototype inferred aggregation from the verified format strings. That is a limited heuristic, not a general modeling standard. Prefer explicit aggregation metadata when available.
- Organizational averages here are unweighted because numerator/denominator or weighting fields were unavailable. Do not present them as weighted enterprise rates.
- Keep numerical measures numerical. Add display-text measures only where a visual needs metric-specific formatting.
- For recurring production work, evaluate supported dynamic format strings instead of turning every matrix value into text. Text display measures affect sorting and export behavior.
- Sort metric groups, metric names, districts, and week labels using actual sort columns. Order separate KPI visuals using the verified KPI sort metadata.
- Hide technical keys and sort columns without deleting them.

### Weekly Time Logic

- Use `FiscalYear`, `FiscalPeriod`, `FiscalWeek`, and `PeriodWeek` for this weekly model.
- Do not mark a weekly calendar as a daily date table without verifying all date-table requirements and existing date variations.
- Preserve organizational and metric filters when moving to the preceding fiscal week.
- Clear only the calendar filters needed for previous-week retrieval. The current prototype clears Calendar and reapplies year/week; a future implementation should explicitly decide how independent date filters should behave.
- Decide and test what happens at fiscal week 1 and fiscal-year boundaries. The current prototype returns blank when the preceding week would be less than 1.
- Define whether "latest week" means the latest calendar week or latest week with loaded actuals. This prototype uses the calendar maximum in the selected period.
- YTD for rates uses an average across applicable weekly results; additive counts use a fiscal-year total through the selected week.
- Align a goal's time horizon with the result being evaluated. A weekly count goal is not automatically a valid target for a YTD sum.
- Use the actual workbook dates. Period 9 here spans 08/21/2026 through 09/17/2026, not a calendar month.
- If weekly measures follow the selected period, avoid fixed captions such as `P9 W1 (W33)`. Period-neutral `Week 1` through `Week 4` were adopted instead.
- Show a real slicer default when measures default to P9. Showing `All` while silently calculating P9 is confusing.

### Performance and Missing Values

- Respect `HigherIsBetter` for both goal status and comparison colors.
- Match each historical cell's color to its own numeric result, not the latest-week result.
- Some lower-is-better thresholds in this workbook are expressed as actual/goal values greater than 1. When calculating a goal/actual ratio, normalize those thresholds to the same convention before comparing them.
- Treat zero as valid where appropriate; for example, zero incidents can be favorable. Do not confuse zero with missing data or divide by zero.
- Return neutral colors for unavailable results and comparisons.
- Keep goals visually neutral. Do not color goal values as though they were actual results.
- Test missing prior weeks, zero prior values, negative growth, lower-is-better metrics, and organization-wide selections.

## Errors Encountered and Fixes

| Symptom | Cause or Finding | Fix and Future Check |
| --- | --- | --- |
| Desktop refused a visual because it detected a UTF-8 BOM | Windows PowerShell 5.1 `Set-Content -Encoding utf8` added a BOM to six repaired visual files | Use UTF-8 without BOM and verify raw bytes, not just decoded JSON. |
| Desktop reported root-level `objects` and `visualContainerObjects` as additional properties | Brace repairs produced syntactically valid JSON with formatting outside `visual` | Place both blocks under `visual`; validate schema and nesting as well as JSON syntax. |
| Several generated visual files failed JSON parsing | Manually constructed, densely nested JSON had missing or extra braces | Use structured serialization for repeated mechanical transformations, or small well-formed patches. Validate immediately instead of repeatedly guessing brace counts. |
| DAX reported that `displayFolder` had invalid syntax | Multiline DAX and measure metadata had the same indentation, so metadata was read as expression text | Indent multiline DAX deeper than metadata; use an actual TMDL parser or Desktop, not only editor diagnostics. |
| KPI Card Text reported an error in Current Week Display | The display measures used a problematic local name, `Value` | Rename to descriptive identifiers such as `DisplayValue`. Seven display measures were corrected. |
| KPI Card Text next reported an error in KPI Goal Text | The goal-text measure used the local identifier `Goal` | Rename to `TargetValue`; the neighboring ratio measure was updated too. The user subsequently reported no errors. |
| KPI values were blank while the matrix contained data | Metric filters previously stored in a visual's formatting/general block were not retained in the saved Desktop definition | Use supported categorical visual-level filters on the real metric key. Apply the same metric filter to value, goal, week, and change components. |
| Report title was clipped; short header cards showed no text | Theme padding consumed much of the available height | Use a textbox for static titles, explicit padding, and adequate visual height. Disable unwanted inherited borders and shadows. |
| Hovering over the help button produced blue text on blue | Only default-state colors had been set; the theme supplied interaction colors | Set explicit `default`, `hover`, and `press` fill/text pairs on open and close buttons. |
| Clicking the help button appeared to do nothing | In Desktop edit mode, a normal click selects a button rather than executing its action | Use Ctrl+click; test the bookmark directly to distinguish action-link problems from overlay visibility problems. |
| Collapsing/drilling the matrix could remove all rows | Mixed-metric groups intentionally return blank and can be suppressed by the visual | Enable `query.queryState.Rows.showAll`, retain a valid hierarchy/expansion state, and keep group labels without inventing totals. |
| Clicking one metric row made the other KPIs show `(Blank)` | Matrix selection cross-filtered the fixed-metric KPI visuals into conflicting metric contexts | Add page-level `NoFilter` interactions from the matrix to every data-bearing KPI component. Keep slicer filtering enabled. |
| A final validation check incorrectly said the help bookmarks had changed | Desktop omitted an explicit `suppressDisplay: false` property when saving | Account for equivalent omitted defaults. Do not overwrite working Desktop-generated state to satisfy an overly strict text-level check. |
| Newly generated text could cover the help overlay | New z-order values exceeded the overlay's stacking level | Place new elements just above their owning visual and below the help overlay; validate both z-order and geometry. |
| Generated P9 selection had the wrong categorical-filter shape | PowerShell flattened an intended nested array | Verify that categorical `In.Values` is a two-dimensional array after serialization. |

The `Value` and `Goal` entries record observed troubleshooting outcomes. Do not generalize that every name matching a DAX function is prohibited. Prefer unambiguous local names and inspect the failing measure's actual parser error.

## Safe TMDL Pattern

For multiline measures in this project's indentation style:

- Measure declaration: one tab.
- Metadata such as `displayFolder`, `formatString`, and `lineageTag`: two tabs.
- DAX body: at least three tabs, with deeper indentation for nested expressions.

```tmdl
	measure 'Current Week Display' =
			VAR MetricFormat = SELECTEDVALUE ( Metrics[NumberFormat], "0.0" )
			VAR DisplayValue = [Current Week Actual]
			RETURN IF ( ISBLANK ( DisplayValue ), BLANK (), FORMAT ( DisplayValue, MetricFormat ) )
		displayFolder: KPI
```

Preserve existing lineage tags. Desktop can reorder measures before columns, collapse one-line expressions, normalize quoted property values, and add identifiers. Those changes are not defects by themselves.

## Safe PBIR Patterns

### Visual Structure and Identifiers

The formatting belongs inside the visual, while positioning, grouping, and visual-level filters belong to the container:

```text
visual.json
  $schema
  name
  position
  parentGroupName / isHidden, when applicable
  visual
    visualType
    query
    expansionStates
    objects
    visualContainerObjects
  filterConfig, when applicable
```

A group uses `visualGroup` instead of `visual`. Preserve existing names and generate a new unique name only for a new object. Check bookmark targets, interactions, query references, formatting selectors, and folder names against the actual IDs.

### Metric Filters and Period Defaults

- Put fixed metric restrictions in the visual's supported `filterConfig`, not an arbitrary formatting property.
- Use the imported metric keys and verify them against source data.
- Keep `SourceRef.Entity` references distinct from query aliases such as `SourceRef.Source` inside a filter's `From`/`Where` structure.
- Native slicer selection state can use `visual.objects.general[].properties.filter.filter`. Verify it survives a Desktop save.
- Categorical membership values must retain their row/value array shape:

```json
"Values": [[{ "Literal": { "Value": "'P9'" } }]]
```

In Windows PowerShell, use an explicit outer array when necessary, then inspect the serialized JSON. A successful serialization call does not prove that the expected nested shape was retained.

### Matrix Interactions

Use the page's existing `visualInteractions` array. Preserve unrelated rules and add one override per target:

```json
{
  "source": "scorecard_matrix",
  "target": "kpi_sales_attainment",
  "type": "NoFilter"
}
```

This is an example using current project IDs, not a rule to copy blindly. When a KPI is split into multiple visuals, protect every data-bearing component. The latest layout has value, goal, neutral week text, and colored change text for each of four KPIs: 16 protected components. Static labels do not need data-filter protection.

### Help Overlay and Bookmarks

- Use a native `actionButton` with a bookmark action rather than a card styled to resemble a button.
- A hidden visual group owns the backdrop, help textbox, and Close button.
- Open and close bookmarks must have opposite group visibility states.
- Set `applyOnlyToTargetVisuals: true` and target only the help group and children.
- Set `suppressData: true` so slicer selections are not captured or reset.
- Display changes must be enabled: `suppressDisplay` is false or omitted, not true.
- Do not capture matrix expansion or active projections in a help bookmark.
- Both the Close button and backdrop can invoke the close bookmark.
- Register bookmark IDs in `definition/bookmarks/bookmarks.json`, and verify every action resolves to a registered bookmark.
- Keep the panel and Close button above the backdrop, and the overlay above report content.
- In Desktop edit mode, execute buttons with Ctrl+click. In normal reading mode, use a normal click.
- Do not assume a bookmark overlay provides browser-style focus trapping or Escape-key dismissal. Test accessibility and keyboard navigation explicitly.

### Theme and Layout

- Override relevant theme defaults locally instead of changing an existing generated base theme without need.
- Explicitly set padding, borders, shadows, and button interaction colors where the design depends on them.
- Reserve separate geometry for a KPI's label, value, goal, and comparison text. Increasing fonts without increasing space can cause clipping.
- Do not rely on an unsupported formatting property that Desktop silently drops. Check the saved definition after reopening and saving.
- Keep group shading full-width, goals neutral, and actual-value colors scoped to the displayed result.
- Fixed widths must fit the visual's usable area, including hierarchy indentation and padding. Recheck after Desktop saves or a layout change.
- Compare screenshots at equivalent canvas scaling. Desktop selection handles and edit-mode visual toolbars are not report design elements.

## Encoding and Structured Editing

For permitted bulk mechanical transformations, use structured objects and an explicit BOM-free encoding. Avoid manual shell string concatenation for nested JSON.

```powershell
$encoding = New-Object System.Text.UTF8Encoding($false)
$json = $definition | ConvertTo-Json -Depth 100
$null = $json | ConvertFrom-Json -ErrorAction Stop
[System.IO.File]::WriteAllText($path, $json, $encoding)
```

This is an encoding/serialization pattern, not a substitute for the coding environment's editing rules. Use the required patch mechanism for manual edits. Inspect transformed files for semantic changes, and remove temporary transformation scripts when they are not intended deliverables.

Validate raw bytes for `EF BB BF` at the start of UTF-8 files. Also decode with strict UTF-8 to catch invalid sequences. `ConvertFrom-Json` alone can accept a BOM that Power BI rejects.

## Validation Gates

### Gate 1: Source and Structure

- Every edited JSON file parses; every edited definition uses the required encoding.
- Root properties and nested properties match the file's declared schema.
- All visual, page, bookmark, and model references resolve.
- Visual names, query references, and identifiers are unique in their required scopes.
- Existing page registration, relative model link, source partitions, and relationships remain intact.
- Visuals fit the canvas; child elements fit their intended arrangements; overlay stacking remains correct.
- Existing user changes are preserved. Inspect a Git diff when Git is available, or compare the intended before/after changes explicitly.

### Gate 2: TMDL and DAX

- Use TMDL deserialization tooling if available; otherwise disclose that syntax checking is limited.
- Check expression/metadata indentation, duplicate measures, column types, qualified field references, and local identifiers.
- Evaluate the actual measures in Desktop or a supported Analysis Services connection when available.
- Test one metric, multiple metrics, one district, all districts, another period, missing observations, and zero denominators.
- Test higher-is-better and lower-is-better thresholds, including zero actuals and goal/result horizon differences.

### Gate 3: Desktop Behavior and Rendering

- Open the PBIP successfully, then refresh imported data as needed.
- Confirm the visible slicer default matches the measures' period logic.
- Change Business Unit, Division, District, and Period; check all KPI components and matrix results.
- Select matrix rows; KPIs must remain populated and keep their assigned metrics.
- Expand/collapse groups and use drill controls; group labels must not disappear unexpectedly.
- Open and close help, including via its backdrop; selections and matrix expansion must remain unchanged.
- Check default, hover, pressed, and relevant disabled button states.
- Verify text fits and remains legible at the user's normal Fit to Page size.
- Save in Desktop, then inspect whether settings survived normalization.

Never equate "No errors found" from the editor with successful TMDL parsing, DAX compilation, data evaluation, or correct rendering. This project repeatedly passed editor checks before Desktop exposed a real problem.

## Troubleshooting Order

1. Capture the full Desktop error and the exact measure or visual name.
2. Fix project-loading issues before investigating DAX or layout.
3. For measure dependency errors, inspect the referenced measure directly. The outer KPI message may only report a failed dependency.
4. For blank cards, check loaded data, fixed metric filters, slicer context, matrix selections, and BLANK guards before changing aggregation logic.
5. For invisible text, check padding and available height before increasing fonts.
6. For button failures, distinguish edit-mode selection from action execution, then check bookmark registration and visibility state.
7. Make one small testable correction and rerun the failing check. Avoid repeated speculative fixes or broad refactoring.

## Environment and Handoff

- In this session, Windows PowerShell was available, but `git`, `node`, and `python` were not on PATH. Do not assume the same tooling in the next environment.
- Read-only Excel COM access was available and was used to verify actual cell values/types without modifying the workbook. Always close the workbook and release Excel resources.
- A locally accessible Analysis Services/TMDL parser or running Desktop engine was not available to the assistant. Runtime verification relied on the user's Desktop feedback.
- Windows font metrics provided a useful estimate for help text wrapping, but not a substitute for a Desktop screenshot.
- If Desktop is open while source files are edited, its older in-memory state can overwrite the changes. Ask the user to close without saving that stale session and reopen the project. Preserve any intentional Desktop work first if needed.
- Final handoffs should list what changed, what checks actually ran, what remains unverified, and the smallest Desktop test needed next.

## Suggested Prompt for a Future Report

> Read `Sample/POWER_BI_REPORT_PLAYBOOK.md` and the supplied design reference. Inspect the current PBIP and actual imported model before editing. Preserve existing source connections, relationships, identifiers, bookmarks, and unrelated changes. Present a concise file-level implementation plan and then implement in small validated steps. Use supported PBIR/TMDL definitions and UTF-8 without BOM. Test filter interactions, metric aggregation, weekly time logic, threshold direction, help bookmarks, and visual layout. Distinguish source checks from Desktop-confirmed behavior, and document any remaining verification requirements.