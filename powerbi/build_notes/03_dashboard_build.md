# 03 - Dashboard Build Specification

**6 pages · one theme · every page titled as a question.**

For each page: the visual list, the fields on each visual, the formatting that matters, and the business purpose the page serves. Build order follows the page order - page 1 establishes the frame that every other page reuses.

---

## Design System

Applied through `theme_inventory_teal.json` (View → Themes → Browse for themes). Apply it before building any visual: a theme cannot override formatting set explicitly on a visual, so anything hand-formatted first will not pick it up.

### Palette

Eight categorical hues, computed rather than chosen by eye. Every adjacent pair was checked for colour-vision deficiency: all eight sit inside one lightness band, clear the chroma floor, hold ΔE ≥ 8 under deuteranopia and tritanopia, and clear 3:1 contrast against the light canvas.

| # | Hex | Reads as |
|---|---|---|
| 1 | `#008C82` | teal - default series |
| 2 | `#C47F15` | amber - second series and accent rule |
| 3 | `#3B6FD4` | blue |
| 4 | `#C43E7A` | magenta |
| 5 | `#55882C` | green |
| 6 | `#8A3FA8` | purple |
| 7 | `#C0392B` | terracotta |
| 8 | `#0090B5` | cyan |

**Status colours are reserved and never reused as series colours:**

| Status | Hex | Reasoning |
|---|---|---|
| Healthy | `#1E7A4C` | good |
| Reorder Now | `#C47F15` | warning |
| Below Safety Stock | `#B3261E` | bad |
| Excess | `#3B6FD4` | informational, not a fault - excess is capital misallocated, not an operational failure |

### Surfaces

| Element | Spec |
|---|---|
| Page canvas | `#F4F7F6` · outspace `#E4EDEA` |
| Cards | White, `#DCE5E2` hairline, 8px radius |
| Ink | `#1A2421` · secondary `#3E4B48` · muted `#6E7C79` |
| Header band | `#06403D`, full width, with a 4px amber `#C47F15` rule beneath |
| KPI value | `#0B5E59` - same family as the header, one step brighter, so the number is where the eye lands |
| Tables | Headers reversed out of `#06403D`; horizontal gridlines only; alternating `#F7FAF9` rows |
| Axes | Gridlines `#EDF2F0` on the value axis only; axis titles off; legends top-aligned, no legend title |
| Text | Never wears a series colour |

### Page frame - identical on all six pages

```
┌──────────────────────────────────────────────────────────────────────┐
│  HEADER BAND  #06403D    "How healthy is our stock?"                 │
│━━━━━━━━━━━━━━━━━━━━━━ 4px amber rule ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━│
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐             │
│  │  KPI   │ │  KPI   │ │  KPI   │ │  KPI   │ │  KPI   │   slicers   │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘             │
│  ┌──────────────────────────┐  ┌──────────────────────────┐         │
│  │        visual            │  │         visual           │         │
│  └──────────────────────────┘  └──────────────────────────┘         │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  INSIGHT BOX  — a measure, not static text                   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  Assumptions Footer — current parameter state, every page            │
└──────────────────────────────────────────────────────────────────────┘
```


---

## Page 1 - Inventory Health Overview

### *"How healthy is our stock?"*

**Business purpose.** The executive landing page. It answers, in one screen, the three questions a director asks about inventory: how much capital is in it, how much of that is wasted, and how many items need someone to do something today. Everything else in the report exists to explain a number on this page.

### Visuals

| # | Visual | Fields | Formatting |
|---|---|---|---|
| 1–7 | KPI cards ×7 | `On Hand Value` · `Total Safety Stock Value` · `Excess Value` · `Excess % of On Hand` · `Inventory Turns` · `Days on Hand` · `SKUs Reorder Now` | White card, hairline, 8px radius. Value `#0B5E59`, 28pt semibold. Currency `£#,##0,K` / `£#,##0,,M`. Label 10pt `#6E7C79` uppercase |
| 8 | Donut - SKU count by status | Legend `Status List[Status]` · Values `SKUs in Status` | Inner radius 60%. Data labels: category + value. Status colours must be set by hand - see the note below |
| 9 | Bar - excess value by category | Y `Dim_SKU[Category]` · X `Excess Value` | Sorted descending. Single teal series. Data labels on, `£#,##0` |
| 10 | Bar - SKU count by ABC | Y `Dim_SKU[ABC]` · X `DISTINCTCOUNT(StockCode)` | Small multiple to the donut |
| 11 | Slicer - Category | `Dim_SKU[Category]` | Dropdown, multi-select |
| 12 | Slicer - ABC class | `Dim_SKU[ABC]` | Tile, single row |
| 13 | Insight box | Text box with the three headline findings | Left-aligned, 12pt, white background |
| 14 | Footer | `Assumptions Footer` measure | 9pt `#6E7C79`, full width |

### Card labels - one deliberate wording decision

The card reads "SKUs needing action", not "SKUs to reorder now". `SKUs Reorder Now` counts Reorder Now plus Below Safety Stock (2,099 at full scope), while the Excel `Summary` sheet's "SKUs to reorder now" uses the narrow definition (107 in the top-500 scope). Two definitions, two labels - rather than silently changing one to match the other.

### The status donut colours

**Theme hues are assigned in category order**, so out of the box "Below Safety Stock" comes out teal and "Healthy" comes out blue - colour that actively contradicts meaning. The model carries a `Status List Colour` measure for this.

**Do it in the UI:** select the donut → Format → Data colours → fx → Format style: Field value → `Status List Colour`.

### Key numbers on this page (3,775 SKUs, default parameters)

| Card | Value |
|---|---|
| On-hand value | £2.0M |
| Safety stock value | £679K |
| Excess value | £371K |
| Excess % | 20.3% |
| Inventory turns | 3.2x |
| Days on hand | 115 |
| SKUs needing action | 2,099 |

Donut: Healthy 1,454 (38.52%) · Below Safety Stock 1,308 (34.65%) · Reorder Now 791 (20.95%) · Excess 222 (5.88%).

---

## Page 2 - ABC-XYZ Segmentation Matrix

### *"Where is the value, and how predictable is it?"*

**Business purpose.** Turns 3,775 undifferentiated SKUs into nine policy cells, and shows leadership where planner attention should go. This is the page that carries the project's central finding - that almost nothing in this portfolio can be automated.

### Visuals

| # | Visual | Fields | Formatting |
|---|---|---|---|
| 1 | Matrix - the 3×3 | Rows `Dim_SKU[ABC]` · Columns `Dim_SKU[XYZ]` · Values `DISTINCTCOUNT(StockCode)`, `Annual Consumption Value` | Background conditional formatting on value, teal→dark gradient. Row/column subtotals on. No word wrap |
| 2 | Pareto - value concentration | X `Dim_SKU[StockCode]` (sorted by `ValueRank`) · Column `Annual Consumption Value` · Line `CumValueShare` | Line on the secondary axis, 0–100%. Constant line at 80%, amber, labelled. Columns teal, no gap |
| 3 | Scatter - CV vs value | X `Annual Consumption Value` · Y `CV` · Details `Dim_SKU[StockCode]` · Legend `Dim_SKU[ABC]` | X axis → Range → Logarithmic scale. Constant lines at CV 0.5 and CV 1.0, labelled X\|Y and Y\|Z. Marker size 4 |
| 4 | Table - policy per segment | `ABC_XYZ`, SKU count, ACV, share of value, recommended policy | Static policy text as a calculated column or a small entered table |
| 5 | Card - X-class SKU count | `DISTINCTCOUNT(StockCode)` filtered to XYZ = X | The headline finding, given its own card |
| 6 | Insight box | Narrative measure | |
| 7 | Slicer - Category | `Dim_SKU[Category]` | |

### The logarithmic axis transforms this page

On a linear X axis, 3,775 SKUs spanning four orders of magnitude of consumption value are crushed against the left edge and the chart says nothing. On a log axis the A, B and C bands separate cleanly and the CV constant lines become readable as policy boundaries.


### Matrix values (3,775 SKUs, partial read)

| Cell | SKUs | Annual consumption value |
|---|---|---|
| A / X | 7 | £91,994 |
| A / Y | 254 | £1,739,349 |
| B / Y | 106 | £103,502 |
| C / Y | 13 | £5,413 |
| X total | 7 | - |
| Y total | 373 | £1,848,265 |

**Only 7 steady SKUs in 3,775.** The Excel top-500 scope returns the same absolute count. This is the number the page exists to deliver.

---

## Page 3 - Service Level Simulator

### *"What does better service cost?"*

**Business purpose.** Converts an argument into a number. Service level is normally debated on opinion; this page prices it, live, in front of the people making the decision. It is the page to demo in an interview.

### Visuals

| # | Visual | Fields | Formatting |
|---|---|---|---|
| 1 | Slider - Service Level | `'Service Level'[Service Level]` | Between-style slider, 0.80–0.995, step 0.005. Top left, prominent |
| 2 | Slider - Lead Time Change | `'Lead Time Change'[Lead Time Change]` | −2 … +4 weeks, step 1 |
| 3 | Slicer - SL Mode | `'SL Mode'[Mode]` | Tile, two options, single select |
| 4 | Slider - Ordering Cost | `'Ordering Cost'[Ordering Cost]` | £20–£100 |
| 5 | Slider - Holding Cost % | `'Holding Cost %'[Holding Cost %]` | 15%–35% |
| 6–9 | Reactive KPI cards ×4 | `Total Safety Stock Value` · `SKUs Reorder Now` · `Reorder Point Inventory Value` · `Excess Value` | Same card style as page 1. Every one moves as the sliders move |
| 10 | Line - safety stock vs service level | X `'Service Level Curve'[Level]` · Y `SS Value at Level` | Teal, 3px, markers on, data labels at each point. Y axis `£#,##0,K` |
| 11 | Column - safety stock by ABC class | X `Dim_SKU[ABC]` · Y `Total Safety Stock Value` | Shows where the extra buffer lands |
| 13 | Headline | `Simulator Headline` measure | Writes its own sentence from the slider position |
| 14 | Headline | `Lead Time Headline` measure | Same idea for the lead-time slider |
| 15 | Footer | `Assumptions Footer` | |

### Two things about the curve

**1. The curve table must be derived from the parameter column.** A hand-typed `0.97` is not bit-identical to `0.8 + 0.005 × 34` in floating point. `TREATAS` matches exact values, so a typed table matches nothing and the chart renders blank, with no error. This was the single highest-risk item in the build.

**2. `SS Value at Level` pins `SL Mode` to Uniform.** In class-based mode the service level comes from ABC rather than the parameter, so `TREATAS` has nothing to change and the curve comes out flat - a perfectly plausible-looking horizontal line.

### The constant line that was removed

A 95% reference line was specified for the curve. Power BI offers constant lines only on the value axis of a line chart with a categorical X, so on this chart it was inert. It was removed rather than left in place looking decorative - the `Simulator Headline` card states the figure in words instead.

### Key outputs

**Excel top-500 scope, uniform mode:** 95% → 98% = +£82,081 (+24.9%) · 95% → 99% = +£136,802 (+41.4%)
**Full 3,775 scope:** uniform 95% = £583,623 vs class-based £679K
**Lead time +2 weeks (top-500):** safety stock +£76,200 (+19.6%), reorder-point inventory +£224,843 - a £301,043 working-capital call

---

## Page 4 - Planner Action List

### *"What do we order today?"*

**Business purpose.** The operational output. Everything upstream exists to produce this list. A buyer should be able to open this page, sort by value, and start raising purchase orders without asking anyone a question.

### Visuals

| # | Visual | Fields | Formatting |
|---|---|---|---|
| 1–4 | KPI cards ×4 | `SKUs Below Safety Stock` · `SKUs Reorder Now (strict)` · `A-Class SKUs Below Safety Stock` · `Suggested Order Value` | The first two must sum to the table's row count - a live self-check on the page |
| 5 | Action table | `StockCode` · `Description` · `ABC_XYZ` · `Dim_SKU[OnHand]` · `Safety Stock Units` · `Reorder Point Units` · `EOQ Units` · `Suggested Order Qty` · `Stock Status` · `Annual Consumption Value` | Sorted by `Annual Consumption Value` descending - highest-value action first. Column widths fixed; numbers right-aligned; totals row off |
| 6 | Buttons ×2 | "Action" / "Excess" | Switch `Action View`; selected state amber |
| 7 | Slicers | `Dim_SKU[ABC]`, `Dim_SKU[Category]`, `Dim_SKU[XYZ]` | |
| 8 | Insight box | Narrative measure | |
| 9 | Note | "HOW TO USE THIS PAGE" | Static text |

### The filter that makes the page work

`Action Filter` is added to the table's Filters on this visual well with the condition *is 1*, and is not placed in the table's columns, so it filters without appearing as a column. The table then shows only Reorder Now and Below Safety Stock rows, and the row count matches the cards above it - which is the page's built-in validation.

This is necessary because `Stock Status` is a measure, not a column, and therefore cannot be placed on a filter pane or a slicer.

## Page 5 - SKU Detail (drill-through)

### *"What is going on with this item?"*

**Business purpose.** Where a planner lands after clicking a SKU on the action list. One item, its full demand history and its complete policy in one frame.

### Visuals

| # | Visual | Fields | Formatting |
|---|---|---|---|
| 1 | Title card | `SKU Detail Title` measure | Prints an instruction when nothing is selected |
| 2 | Slicer - Choose a SKU | `Dim_SKU[StockCode]` | Dropdown, Single select ON |
| 3 | Line - 52-week demand | X `Dim_Week[WeekStart]` · Y `Units Sold` · plus `Avg Weekly Demand (all weeks)` and `Avg Plus 1 SD (all weeks)` | Demand teal 2px; the two reference lines flat, dashed, muted |
| 4–11 | Cards ×8 | `ABC (selected)` · `XYZ (selected)` · `CV (selected)` · `Effective Lead Time` · `Safety Stock Units` · `Reorder Point Units` · `EOQ Units` · `Dim_SKU[OnHand]` | Compact, two rows of four |
| 12 | Card - status | `Stock Status` | Text card |
| 13 | Note | "HOW TO READ THIS PAGE" | Explains the single-SKU grain and the link to the page-3 sliders |
| 14 | Drill-through well | `Dim_SKU[StockCode]` | Enables right-click drill from any page |

### Blank cards on first open

Originally the page opened with the title reading `SKU  -  |  |` and seven cards reading (Blank). Every card is a single-SKU figure - the `… (selected)` measures are `SELECTEDVALUE` and the policy measures are wrapped in `HASONEVALUE`. With nothing selected they correctly returned blank.

**The dangerous card was the one that showed a number.** `[CV]` had no single-SKU guard, so it was quietly reporting the portfolio CV of 0.38 beside seven blanks - which looks like data.

Four changes:

1. **`SKU Detail Title` is `HASONEVALUE`-guarded** and prints an instruction instead of an empty template.
2. **New measure `CV (selected)`** = `IF ( HASONEVALUE ( Dim_SKU[StockCode] ), [CV] )`, and the CV card rebound to it. `[CV]` itself is unchanged - pages 1–4 still need the portfolio aggregate.
3. **Slicer set to Single select.** A detail page should never be able to sum two SKUs together.
4. **A SKU is saved as the opening selection** - `10002` (INFLATABLE POLITICAL GLOBE, C/Z, CV 2.28, lead time 6 wk, SS 109, ROP 201, EOQ 787, on hand 198 - sitting just below its reorder point). The page opens populated; any SKU can still be chosen.


### Reference lines must be computed over `ALL(Dim_Week)`

`Avg Weekly Demand` and `Avg Plus 1 SD` plotted as series collapse onto the `Units Sold` line, because at weekly grain the average of one week is that week. The `(all weeks)` variants wrap the calculation in `CALCULATE(…, ALL(Dim_Week))` so they render as flat reference lines under a spiky demand series - which is what the chart was for.

---

## Page 6 - Excel Reconciliation

### *"Do the two models agree?"*

**Business purpose.** The page reads the Excel workbook from disk on every refresh and grades the semantic model against it.

### Visuals

| # | Visual | Fields |
|---|---|---|
| 1 | Verdict card | `Reconciliation Verdict` - checks the parameter state first and refuses to grade under non-default assumptions |
| 2–10 | Mismatch-count cards ×9 | `ABC Mismatches` · `XYZ Mismatches` · `EOQ Mismatches` · `SS Mismatches (Excel ABC)` · `ROP Mismatches (Excel ABC)` · `On Hand Mismatches` · `Excel Scope SKU Count` · `Grid Check` · `SS Mismatches` |
| 11 | Comparison table | `StockCode` · live SS / ROP / EOQ / ABC / XYZ beside `SS_Excel` / `ROP_Excel` / `EOQ_Excel` / `ABC_Excel` / `XYZ_Excel` |
| 12 | Slicer - Scope | `Dim_SKU[In Excel Scope]` - switch between the shared 500 and the full 3,775 |
| 13 | Note | Explains the ABC scope difference and the `21908` floating-point tie |

### Required parameter state

| Parameter | Value |
|---|---|
| SL Mode | Class-based (A 98 / B 95 / C 90) |
| Lead Time Change | 0 |
| Ordering Cost | £50 |
| Holding Cost % | 25% |

**The verdict measure checks these before grading.** A reconciliation that silently grades under the wrong assumptions is worse than no reconciliation.

### Result

> **MATCH ON POLICY** - across all 500 shared SKUs the two models agree on XYZ class, EOQ, and, using the workbook's own ABC class, on safety stock and reorder point. The only residual is 1 SKU where the simulated on-hand snapshot lands exactly on a half unit and the two engines break the tie in opposite directions.

Full analysis: [`../../docs/validation.md`](../../docs/validation.md).

---

## Cross-Page Build Notes

### Action / Excess toggle

Page 4 switches between the order list and the excess list through the disconnected `'Action View'[View]` table (values *Action* and *Excess*). `Action Filter` reads the current selection and returns 1 for the rows that belong in that view.

### Drill-through

Configured on page 5 with `Dim_SKU[StockCode]` in the drill-through well. Right-click any SKU on pages 2 or 4 → Drill through → SKU Detail. Keep "Keep all filters" on so the detail page inherits the slider state - a SKU's safety stock must be shown at the same service level the user was looking at.

### Interactions

**Format → Edit interactions**, page by page. Defaults are usually wrong: a KPI card should generally not be cross-filtered by a click on a donut slice, because a "portfolio on-hand value" card that silently becomes "on-hand value of the Healthy segment" is a number that lies without changing its label.

### Accessibility

- All eight palette hues validated for deuteranopia and tritanopia at ΔE ≥ 8.
- Status is never encoded by colour alone - every status appears as text in the table and in the donut labels.
- Alt text set on all chart visuals.
- Tab order set per page: header → KPI cards → main visual → slicers.

## Page Build Checklist

- [ ] Theme applied before any visual formatting
- [ ] Every page titled as a question, in the `#06403D` header band with the amber rule
- [ ] Every page carries an insight box driven by a measure, not static text
- [ ] `Assumptions Footer` on every page
- [ ] Page 1 donut colours set via fx → Field value → `Status List Colour` in the UI
- [ ] Page 2 scatter X axis logarithmic - verified on screen, not assumed
- [ ] Page 2 constant lines at CV 0.5 and CV 1.0, labelled
- [ ] Page 3 curve table derived from the parameter column; `SS Value at Level` pins SL Mode to Uniform
- [ ] Page 4 `Action Filter` applied as a visual-level filter `= 1` and dropped from the projection
- [ ] Page 4 card counts sum to the table row count
- [ ] Page 5 slicer Single select, with a SKU saved as the opening selection
- [ ] Page 5 reference lines computed over `ALL(Dim_Week)`
- [ ] Page 6 verdict reads MATCH ON POLICY at default parameters
- [ ] Edit interactions reviewed on every page
- [ ] Alt text on every chart
- [ ] Export to PDF for `inventory_optimization.pdf`
