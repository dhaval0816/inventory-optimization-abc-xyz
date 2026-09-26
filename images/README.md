# `images/` - Screenshot Specification

What each image must show, how to capture it, and what it is there to prove. A portfolio screenshot has one job: make a recruiter who will not open the .pbix believe the .pbix works.

---

## Capture standard - applies to every image

| Setting | Value |
|---|---|
| Resolution | 1920 × 1080 minimum. GitHub scales down cleanly; it cannot scale up |
| Format | PNG for stills, GIF for the simulator clip |
| Power BI view | View → Page view → Fit to page, Desktop maximised, no side panels open |
| Chrome | No Windows taskbar, no browser tabs, no file paths, no other application visible |
| Parameter state | Defaults: SL Mode Class-based (A 98 / B 95 / C 90) · Lead Time Change 0 · Ordering Cost £50 · Holding Cost 25% |
| Data state | No slicer filtered unless the shot is specifically about filtering |
| File size | Under 1 MB per PNG, under 5 MB for the GIF - GitHub renders slow images badly |
| Tool | Windows Snipping Tool / Win+Shift+S for stills · ScreenToGif for the clip |

**Three things that must never appear in a screenshot:** a `(Blank)` card, an error triangle, or a visual still showing sample data. Any one of them undoes the credibility the rest of the repository is trying to build.

---

## `01_inventory_health.png` - the hero image

**Source:** Report page 1 - *"How healthy is our stock?"*

**Purpose.** The image at the top of the README, and for most visitors the only one they will look at. It has to communicate scale, competence and a business message in about two seconds.

**Layout that must be visible**

```
┌──────────────────────────────────────────────────────────────┐
│  How healthy is our stock?                          [header] │
│━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━│
│ ┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐    │
│ │ £2.0M││ £679K││ £371K││ 20.3%││ 3.2x ││ 115  ││ 2,099│    │
│ │ ON   ││SAFETY││EXCESS││EXCESS││TURNS ││ DAYS ││ NEED │    │
│ │ HAND ││STOCK ││      ││  %   ││      ││  OH  ││ACTION│    │
│ └──────┘└──────┘└──────┘└──────┘└──────┘└──────┘└──────┘    │
│ ┌────────────────────────┐  ┌────────────────────────────┐  │
│ │  Donut: SKUs by status │  │  Bar: excess by category   │  │
│ │  (four status colours) │  │  (sorted descending)       │  │
│ └────────────────────────┘  └────────────────────────────┘  │
│ ┌──────────────────────────────────────────────────────┐    │
│ │ INSIGHT: 2,099 of 3,775 SKUs need action today…      │    │
│ └──────────────────────────────────────────────────────┘    │
│ Simulated inputs: unit cost = median price × 0.60 …  [footer]│
└──────────────────────────────────────────────────────────────┘
```

**Insights it must display:** £2.0M on hand · £679K safety stock · £371K excess (20.3%) · 3.2x turns · 115 days on hand · 2,099 SKUs needing action · the four-way status mix with Below Safety Stock at 1,308 (34.65%).

**Checks before you keep it:** status donut colours semantically correct (red = Below Safety Stock, not teal) · insight box showing real numbers, not placeholder text · assumptions footer legible.

---

## `02_abc_xyz_matrix.png`

**Source:** Report page 2 - *"Where is the value, and how predictable is it?"*

**Purpose.** Proves segmentation competence, and carries the project's central finding - only 7 of 3,775 SKUs are steady enough to automate.

**Must be visible**

- The 3 × 3 matrix, ABC rows × XYZ columns, with SKU count and annual consumption value, conditional formatting on, subtotals on.
- The Pareto chart with the 80% constant line.
- The scatter on a logarithmic X axis, with the CV 0.5 and CV 1.0 constant lines labelled.
- The X-class count card reading 7.

**Insights it must display:** A/X 7 SKUs (£91,994) · A/Y 254 SKUs (£1,739,349) · X total 7 · Y total 373 SKUs (£1,848,265) · Pareto crossing 80% where A ends.

> **Verify the scatter axis on screen before capturing.** If the X axis is linear, every point is crushed against the left edge and the chart proves nothing. The log toggle is X-axis → Range → Logarithmic scale - the JSON property does not work.

---

## `03_service_level_simulator.gif` - the one that gets attention

**Source:** Report page 3 - *"What does better service cost?"*

**Purpose.** A still image cannot show interactivity. This clip is what convinces a reviewer the report is a tool, not a picture of one.

**Capture script**

| Time | Action |
|---|---|
| 0.0 – 1.5s | Hold still on the default state, service level 95% |
| 1.5 – 4.0s | Drag the Service Level slider slowly 95% → 99%. Let the cards and the curve redraw |
| 4.0 – 5.5s | Pause on 99% so the new numbers and the insight sentence are readable |
| 5.5 – 7.5s | Drag the Lead Time Change slider 0 → +2. Pause |
| 7.5 – 9.0s | Return both to default and hold |

**Specification:** 8–12 seconds · 10–15 fps · loop on · under 5 MB · no cursor jitter - move deliberately, one slider at a time.

**What must be readable while it moves:** the `Total Safety Stock Value` card, the service-level curve redrawing, and the insight box rewriting its own sentence. The last one is the point of the clip - it shows the narrative is measure-driven, not a static caption.

**Numbers that should appear:** 95% → 98% = +£82,081 (+24.9%) · 95% → 99% = +£136,802 (+41.4%) · lead time +2 weeks = +£76,200 safety stock.

> Record at a steady pace. A fast drag produces a clip nobody can read, and a clip nobody can read is worse than a still.

---

## `04_action_list.png`

**Source:** Report page 4 - *"What do we order today?"*

**Purpose.** Proves the project produces an operational output, not a dashboard. This is the image that separates an analysis from a deliverable.

**Must be visible**

- The four KPI cards, with counts that sum to the visible table rows - the page's built-in self-check.
- The action table, sorted by annual consumption value descending, showing StockCode, Description, ABC_XYZ, On Hand, Safety Stock, ROP, EOQ, Suggested Order Qty and Stock Status.
- At least 12–15 rows, so it reads as a real list.
- Both Action / Excess buttons, with the active state visible.

**Insights it must display:** the top rows are high-value items with a real suggested order quantity · only *Reorder Now* and *Below Safety Stock* rows appear (the `Action Filter` is working) · `A-Class SKUs Below Safety Stock` visible on a card.

> The status column is not conditionally coloured - that formatting does not render in this report file format and the dead formatting was removed rather than left in place. Do not retouch the screenshot to add it.

---

## `05_data_model.png`

**Source:** Power BI Model view

**Purpose.** For the technical reviewer. A clean star schema is the fastest way to signal that the author understands data modelling rather than just visual building.

**What the committed image shows**

- `Fact_WeeklyDemand` in the centre with `Dim_Week` on its left and `Dim_SKU` on its right, both relationship lines showing 1 → \* cardinality and a single-direction filter arrow pointing into the fact table.
- Only those three tables. The shot was taken on its own Model view layout tab, built with right-click → **Add related tables** on `Fact_WeeklyDemand`, so the star schema reads cleanly instead of competing with the ten disconnected tables.
- The parameter tables (`Service Level`, `Lead Time Change`, `Ordering Cost`, `Holding Cost %`, `SL Mode`, `Status List`, `Segment Policy`, `Service Level Curve`, `Action View`) and the `_Measures` host table live on the default **All tables** tab. They are disconnected by design - nothing joins them to the fact table - which is exactly why they are absent from a *related tables* layout.
- `Dim_Week` carries **no** date-table marker. Power BI's *Mark as date table* requires a contiguous daily calendar and this one is weekly, so the marker was refused rather than faked. `docs/assumptions.md` records that.

**Capture tips:** build a separate layout tab and use *Add related tables* rather than fighting the default auto-layout. Collapse long field lists so the table cards stay compact. Capture at 1920px wide so every table and column name is legible.

**What it proves:** a star schema, not a web · single-direction relationships · a fact table joined on exactly two keys · parameters deliberately outside the model.

---

## `06_excel_policy_sheet.png`

**Source:** `inventory_policy_calculator.xlsx`, sheet `SKU_Policy`, range A1:Z26 (Home → Copy → Copy as Picture).

**Purpose.** Shows the policy sheet as it looks in the workbook: ABC and XYZ class, target service level, Z-score, safety stock, reorder point, EOQ, on hand, weeks of cover and excess for the top SKUs. Open the workbook and click any cell on this sheet to see the formula behind it.

---

## `07_power_query_applied_steps.png` - proof the ETL was clicked, not coded

**Source:** Power BI Desktop → Home → Transform data → query `Fact_WeeklyDemand`

**Purpose.** Shows that the ETL was built by clicking the ribbon, not by writing M. This is the image that backs up the claim in the README - and the step names are left exactly as Power Query generated them, because a default name like `Unpivoted Other Columns` is itself the evidence that a ribbon button produced it.

**What the committed image shows**

- The **Applied Steps** pane on the right, with all eleven steps of `Fact_WeeklyDemand` visible and named by Power Query itself:

  `WS` · `Removed Columns` · `Appended Query` · `Pivoted Column` · `Replaced Value` · `Unpivoted Other Columns` · `Renamed Columns` · `Changed Type` · `Merged Queries` · `Expanded Weekly_Sold` · `Replaced Value1`

  Every one of those names is the label Power Query writes when you use the matching ribbon command. `Pivoted Column`, `Unpivoted Other Columns`, `Merged Queries` and `Expanded Weekly_Sold` in particular are the steps a coded solution would have written by hand.
- The **Queries** pane on the left with the real query list: the two parameters `SourceFile` and `ExcelModelFile`, the source queries `Sales_2009_2010` and `Sales_2010_2011`, the staging queries `Sales_Raw`, `Weekly_Sold`, `Zero_Week_Seed`, `SKU_Description`, `SKU_Attributes` and `Excel_SKU_Policy`, and the three loaded tables `Fact_WeeklyDemand`, `Dim_SKU` and `Dim_Week`.
- No **Advanced Editor** window. It was never needed.

**On the one typed formula.** `Sales_Raw` contains a single `Added Custom` step holding `Date.StartOfWeek([InvoiceDate], Day.Monday)`. It is typed, it is named in the README and in `powerquery/power_query_steps.md`, and the reason is documented there: the ribbon's *Date → Week → Start of Week* button always returns Sunday-based weeks regardless of query locale, and this project needs Monday weeks. It sits in `Sales_Raw`, not in `Fact_WeeklyDemand`, so it is not in this screenshot - a reviewer who wants to see it should select `Sales_Raw`.

**Optional second shot:** `Sales_Raw` selected with the formula bar visible on the `Added Custom` step, so the one typed formula in the whole project can be read directly.

---

## Optional extras worth adding

| File | Source | Why |
|---|---|---|
| `08_reconciliation.png` | Report page 6 | The verdict card reading MATCH ON POLICY with the zero-mismatch cards. Almost no portfolio project has a reconciliation page - this one image does a lot of work |
| `09_sku_detail.png` | Report page 5 | The 52-week demand line with flat average and average+1σ reference lines, with a SKU selected |

---

## Pre-publication checklist

- [ ] All seven images at 1920 × 1080 or better
- [ ] `07` shows Power Query's own default step names - no renaming that could hide where a step came from
- [ ] No `(Blank)` cards, no error triangles, no sample data
- [ ] No file paths, taskbars, browser chrome or personal information
- [ ] Parameters at documented defaults in every shot
- [ ] Numbers in the images match the numbers in the README - a mismatch is the fastest way to lose a reader's trust
- [ ] Status colours semantically correct in `01`
- [ ] Scatter axis confirmed logarithmic in `02`
- [ ] GIF under 5 MB and readable at normal speed
- [ ] Formula bar visible and showing a real formula in `06`
- [ ] Every image referenced from `README.md` and rendering on GitHub (check the published page, not the local preview)
