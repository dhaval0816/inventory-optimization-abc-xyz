# 00 - Build the Model Without Writing M Code

**Rebuild the entire Power BI semantic model and report from ribbon clicks, from an empty Power BI Desktop file to a working 6-page report.**

This is the primary build path for the project. Follow it end to end and you will reproduce the published figures exactly.

> **Why there is no `01` in this folder.** There is no Power Query code to document. Every transformation is a ribbon click, recorded in [`../../powerquery/power_query_steps.md`](../../powerquery/power_query_steps.md) and summarised in Part 2 below.

---

## The design decision behind this file

**Power Query does only ribbon-clickable work. Everything derived moves to DAX calculated columns.**

**The Advanced Editor is never opened, and exactly one *Custom Column* formula is typed in the whole build** - the Monday week-start in Part 2, step 17, for the reason given there. Power Query stores each click as a step internally; every other one of those steps comes from a menu.

Where the ribbon runs out, the work moves to DAX rather than to hand-written M:

| Need | Why not Power Query | DAX instead |
|---|---|---|
| Running total for ABC | The ribbon has no running-total command - it would mean writing M, and unbuffered M re-reads the source once per row | One calculated column, computed once at refresh over an in-memory table |
| Rounding that matches Excel | The ribbon's *Round* uses banker's rounding (round-half-to-even); Excel's `ROUND` rounds half away from zero | DAX `ROUND` matches Excel, so the simulated on-hand agrees with the workbook |
| Simulated lead time / on hand / unit cost | Needs `MOD`, lookup and branching logic | Calculated columns with the logic visible in the model |

**The principle in one line:** Power Query shapes the data; DAX derives from it. That split is what keeps the ETL to ribbon commands and a single typed formula.

---

## Prerequisites

| | |
|---|---|
| Software | Power BI Desktop 2.157 or later |
| Data | `data/raw/online_retail_II.xlsx` - see [`../../data/raw/README.md`](../../data/raw/README.md) |
| Expected build time | 3–4 hours including the first refresh (~6 minutes) |
| Reference | Row counts to verify against: [`../../docs/data_quality.md`](../../docs/data_quality.md) |

---

## Part 1 - Data Load

### 1.0 Set the locale first

**File → Options and settings → Options → Current File → Regional Settings → Locale for import: English (United Kingdom).** This is a UK retailer, so `InvoiceDate` must parse as dd/mm rather than mm/dd. The locale affects date *parsing* only - it does not decide which day a week starts on (see Part 2, step 17).

### 1.1 Connect and stage

| # | Click |
|---|---|
| 1 | Home → Get data → Excel workbook → select `online_retail_II.xlsx` |
| 2 | In the Navigator tick both sheets: `Year 2009-2010`, `Year 2010-2011` |
| 3 | Transform Data - not *Load*. Nothing enters the model until it is clean |
| 4 | On each query: Home → Use First Row as Headers |
| 5 | Right-click each → untick Enable load. They become staging queries |

### 1.2 Append and de-duplicate

| # | Click | Rows |
|---|---|---|
| 6 | Home → Append Queries as New → combine both sheets → rename `Sales_Raw` | 1,067,371 |
| 7 | On the `Year 2009-2010` staging query: `InvoiceDate` → Date/Time Filters → Before `01/12/2010` - removes the 1–9 Dec 2010 overlap duplicated across both sheets | `Sales_Raw` = 1,044,848 |

**Do not skip step 7.** Nine days of December 2010 appear on both worksheets - 22,523 duplicated rows, sitting *inside* the 52-week modelling window. A plain append double-counts them, inflating the mean and σ of every SKU that sold in that period, and it does so in a way that looks entirely plausible on a dashboard.

### 1.3 Type, then trim

| # | Click |
|---|---|
| 8 | Set types: `Invoice` Text · `StockCode` Text · `Description` Text · `Quantity` Whole Number · `InvoiceDate` Date/Time · `Price` Decimal |
| 9 | Home → Remove Columns: drop `Customer ID` and `Country` - unused, and removing them early cuts refresh time on 1M rows |

> **Detect Data Type** typically types `Invoice` as a whole number, which silently breaks the cancellation filter in step 10. Set it back to Text.

---

## Part 2 - Data Transformation

Full click-by-click detail, with the row count after every step: [`../../powerquery/power_query_steps.md`](../../powerquery/power_query_steps.md). Summary:

| # | Step | Ribbon path | Rows |
|---|---|---|---|
| 10 | Remove cancellations | `Invoice` → Text Filters → Does Not Begin With `C` | 1,025,683 |
| 11 | Remove `Quantity` ≤ 0 | Number Filters → Greater Than `0` | 1,022,290 |
| 12 | Remove `Price` ≤ 0 | Number Filters → Greater Than `0` | 1,019,654 |
| 13 | Normalise `StockCode` | Transform → Format → Trim, then UPPERCASE | - |
| 14 | Keep 5-digit product codes | Add Column → Extract → First Characters (5) → type Whole Number → Remove Errors → delete helper column | 1,014,945 |
| 15 | Remove invoice `541431` | Filter - 74,215-unit line whose credit note was already removed | 1,014,944 |
| 16 | Add `LineRevenue` | Select `Quantity` + `Price` → Add Column → Standard → Multiply | - |
| 17 | Add `WeekStart` | Add Column → Custom Column → name `WeekStart`, formula `Date.StartOfWeek([InvoiceDate], Day.Monday)` → type Date · verify with Add Column → Date → Day → Name of Day = Monday only, then delete the helper | - |
| 18 | Keep 52 complete weeks | `WeekStart` → Date Filters → Between `06/12/2010` … `28/11/2011` | 500,376 |
| 19 | Group to SKU × week | Transform → Group By (Advanced): `StockCode` + `WeekStart`; Sum `Quantity` → `Units`, Sum `LineRevenue` → `Revenue` | 96,033 |

**Step 14 is the useful trick.** "Keep rows where the first five characters are numeric" normally needs a `try … otherwise` expression. Extracting five characters, converting to a number, and clicking Remove Errors does the same job with three ribbon commands - the failed conversions *are* the filter.

**Step 17 is the one typed formula, and the reason is worth knowing.** The obvious command is *Add Column → Date → Week → Start of Week*, and it is wrong here: it calls `Date.StartOfWeek` without the optional first-day argument, which defaults to **Sunday** in M. The query locale does not override it. On this dataset a Sunday week drops the 6–12 Dec 2010 week out of the 52-week window entirely and moves every σ, safety stock and ABC class in the model. The *Custom Column* dialog with `Day.Monday` is the correct fix, and the *Name of Day* check is what proves it took.

### 2.1 The zero-fill - three clicks that make the model correct

A SKU that sold nothing in week 14 currently has no row for week 14. Compute σ on the rows that exist and you measure the variability of *selling weeks*, not of *demand* - understating σ and understating safety stock on exactly the intermittent items that need the buffer.

| # | Click | Result |
|---|---|---|
| 20 | Reference `Weekly_Sold` → rename `Fact_WeeklyDemand`; remove `Revenue` | |
| 21 | Select `WeekStart` → Transform → Pivot Column, Values = `Units`, Advanced → Don't Aggregate | 3,775 rows × 52 week columns, gaps are `null` |
| 22 | Select all 52 week columns → Transform → Replace Values: `null` → `0` | Gaps become real zeros |
| 23 | With them still selected → Transform → Unpivot Columns; rename `Attribute`→`WeekStart`, `Value`→`Units`; set types | 196,300 rows |
| 24 | Merge Queries back to `Weekly_Sold` on `StockCode` + `WeekStart` (Left Outer) → expand `Revenue` → replace `null` with `0` | |

**Use *Don't Aggregate*, never the default Sum.** Sum would silently collapse any duplicate `StockCode` + `WeekStart` pair. After step 19 duplicates should be impossible - and *Don't Aggregate* turns that assumption into a test that fails loudly instead of hiding a defect behind a plausible number.

**Verify before moving on:** 196,300 rows · 3,775 distinct SKUs · 52 distinct weeks · 96,033 rows with `Units` > 0 · 100,267 rows with `Units` = 0 (51%). If the zero count is 0, the round trip did not take.

### 2.2 `Dim_SKU` - attributes only

| # | Click |
|---|---|
| 25 | Reference the cleaned transaction query → rename `Dim_SKU` |
| 26 | Group By (Advanced): `StockCode` + `Description`, new column `n` = Count Rows |
| 27 | Sort `StockCode` ascending, then `n` descending → select `StockCode` → Remove Duplicates → delete `n` |
| 28 | From the transaction query, Group By `StockCode` → `MedianPrice` = Median of `Price`; Merge into `Dim_SKU` |

Median, not mean - a single discounted wholesale line would drag a mean well below the item's normal selling price, and unit cost derives from this figure.

> **Fallback** if the sort is not respected across a refresh (Power Query does not guarantee sort stability through Remove Duplicates): Group By `StockCode` → Max of `Description`. Description drives no calculation, so either rule is acceptable - but state which one you used.

### 2.3 Stop here

**Do not build `UnitCost`, `LeadTimeWeeks`, `OnHand`, `Category`, `ABC`, `XYZ` or cumulative share in Power Query.** All 17 derived attributes are DAX calculated columns, specified in [`02_model_and_dax.md`](02_model_and_dax.md). Column names match the measure references exactly, so all 53 measures work unchanged.

**Home → Close & Apply.** First refresh ≈ 6 minutes.

---

## Part 3 - Calculated Columns and Tables

Create these in Model view → Table tools → New column on `Dim_SKU`, in this order - later columns depend on earlier ones.

| Order | Column | Depends on |
|---|---|---|
| 1 | `UnitCost` | `MedianPrice` |
| 2 | `AnnualUnits` | `Fact_WeeklyDemand` |
| 3 | `AvgWeekly`, `StdWeekly` | `AnnualUnits`, fact |
| 4 | `CV` | `AvgWeekly`, `StdWeekly` |
| 5 | `LeadTimeWeeks`, `WeeksOfCover`, `OnHand` | `StockCode`, `AvgWeekly` |
| 6 | `Category` | `Description` |
| 7 | `AnnualConsumptionValue` | `AnnualUnits`, `UnitCost` |
| 8 | `ValueRank`, `CumValueShare` | `AnnualConsumptionValue` |
| 9 | `ABC` | `CumValueShare` |
| 10 | `XYZ`, `ABC_XYZ` | `CV`, `ABC` |
| 11 | `In Excel Scope`, `OnHand Check` | `ValueRank` |

Full DAX for every one: [`02_model_and_dax.md`](02_model_and_dax.md).

`Dim_Week` comes from Power Query: reference `Weekly_Sold`, keep `WeekStart`, remove duplicates, then add Week No (Index Column), WeekEnd (End of Week), Month and Quarter from the Date menu. Mark it as the date table afterwards.

**Three details that cost time if missed:**

1. **`StdWeekly` needs context transition** - `CALCULATE ( STDEVX.S ( VALUES ( Dim_Week[WeekStart] ), CALCULATE ( SUM ( Fact_WeeklyDemand[Units] ) ) ) )`. Without the inner `CALCULATE` the row context does not become a filter context and every SKU returns the portfolio σ.
2. **`CumValueShare` needs a tie-breaker on `StockCode`** - two SKUs with identical annual value would otherwise each count the other's value in the running total, pushing cumulative share past 100% and mis-classifying both. (The Excel model solved the same problem with a `ROUND(value, 2)` rank helper.)
3. **`Category` pads the description with spaces** before matching, so keywords match whole words only. Without it *PAINTING* matches `TIN` and *FLIGHT* matches `LIGHT`.

### What-if parameters

**Modeling → New parameter → Numeric range**, one at a time. Each creates a disconnected table, a column and a `SELECTEDVALUE` measure automatically.

| Parameter | Min | Max | Step | Default |
|---|---|---|---|---|
| Service Level | 0.80 | 0.995 | 0.005 | 0.95 |
| Lead Time Change | −2 | 4 | 1 | 0 |
| Ordering Cost | 20 | 100 | 10 | 50 |
| Holding Cost % | 0.15 | 0.35 | 0.05 | 0.25 |

Plus a small `SL Mode` table (Modeling → New table, `DATATABLE`) with two rows: `Class-based (A 98% / B 95% / C 90%)` and `Uniform (slider)`. `Status List`, `Action View` and `Segment Policy` are built the same way.

> **`Service Level Curve` must be a DAX table derived from the parameter column** - `SELECTCOLUMNS` over `'Service Level'`, not typed literals. A hand-typed `0.97` is not bit-identical to `0.8 + 0.005 × 34` in floating point, so `TREATAS` would match nothing and the curve would render blank.

---

## Part 4 - Relationships

**Model view.** Drag to create; double-click each to verify.

| From | To | Cardinality | Direction | Active |
|---|---|---|---|---|
| `Dim_SKU[StockCode]` | `Fact_WeeklyDemand[StockCode]` | One to many (1:*) | Single | Yes |
| `Dim_Week[WeekStart]` | `Fact_WeeklyDemand[WeekStart]` | One to many (1:*) | Single | Yes |
| `Excel_SKU_Policy[StockCode]` | - | merged into `Dim_SKU` in Power Query, not related | - | - |
| Parameter tables | - | Disconnected by design | - | - |

**Three rules, and the reason for each:**

1. **Single direction, always.** Bi-directional filtering on a star schema creates ambiguous filter paths and gives a Power BI model its characteristic "the number changed and I don't know why" behaviour. Single direction is a deliberate constraint, not an oversight.
2. **Parameter tables stay disconnected.** They are read with `SELECTEDVALUE`, not through a relationship. A related parameter table would filter the fact and reduce the data to whatever the slider is pointing at - which is exactly the opposite of what a what-if parameter is for.
3. **Mark `Dim_Week` as a date table** - Table tools → Mark as date table → `WeekStart`. Time intelligence is unreliable without it, and Power BI's auto date/time hierarchies bloat the model.

**Hide from report view:** every key column (`Fact_WeeklyDemand[StockCode]`, `Fact_WeeklyDemand[WeekStart]`), every helper column, and the whole `_Measures` table's placeholder column. A field list that only offers fields a report author should use is part of the model's design.

---

## Part 5 - Measures

Create an empty table (Home → Enter data → single column, load, hide the column) named `_Measures` and build all measures there. Grouped into display folders: Demand · Policy · Inventory · Scenario.

All 84 measures with full DAX: [`02_model_and_dax.md`](02_model_and_dax.md).

**Two things to know before you start:**

**1. Per-SKU measures do not sum.** Every portfolio total must iterate:

```dax
Total Safety Stock Value =
SUMX (
    VALUES ( Dim_SKU[StockCode] ),
    [Safety Stock Units] * CALCULATE ( SELECTEDVALUE ( Dim_SKU[UnitCost] ) )
)
```

`SUM([Safety Stock Units])` would return a number - a plausible, meaningless one, because at portfolio grain there is no single σ or lead time to evaluate. This is the most common cause of a Power BI inventory model that looks right and is wrong.

**2. `Stock Status` is a measure, so it cannot go on a legend, slicer or filter.** Work around it with disconnected helper tables:

| Need | Helper |
|---|---|
| Status donut on page 1 | `Status List` table + `SKUs in Status` measure |
| Action-list filter on page 4 | `Action View` table + `Action Filter` measure applied as a visual-level filter `= 1` |

---

## Part 6 - Visualization Setup

Page-by-page specification - every visual, every field, every format setting: [`03_dashboard_build.md`](03_dashboard_build.md).

### 6.1 Theme first

**View → Themes → Browse for themes** → `theme_inventory_teal.json`.

Apply the theme before building visuals. A theme cannot override formatting set explicitly on a visual, so anything formatted by hand first will not pick it up.

### 6.2 Page skeleton

Every page uses the same frame, so the report reads as one object:

| Element | Spec |
|---|---|
| Header band | `#06403D`, full width, page title phrased as a question |
| Accent rule | 4px amber `#C47F15` textbox directly beneath the header |
| KPI cards | Top row, white, `#DCE5E2` hairline, 8px radius, value in `#0B5E59` |
| Insight box | One per page - a measure, not static text, so it composes its own sentence from the current slider state |
| Slicers | Right-aligned or top, consistent position across pages |

### 6.3 Build order

1. Page 1 - Inventory Health Overview (7 KPI cards, status donut, excess by category)
2. Page 2 - ABC-XYZ Segmentation (matrix, Pareto, log-scale scatter, policy table)
3. Page 3 - Service Level Simulator (sliders, reactive cards, service-level curve)
4. Page 4 - Planner Action List (filtered table, Action / Excess toggle)
5. Page 5 - SKU Detail (drill-through, single-select slicer)
6. Page 6 - Excel Reconciliation (mismatch cards, verdict)

### 6.4 Things that are easy to miss

| Issue | Fix |
|---|---|
| Logarithmic X axis on the Page 2 scatter | Format → X-axis → Range → Logarithmic scale. Check it on screen after saving |
| Constant line on the service-level curve | Power BI only offers constant lines on the value axis of a line chart, so state the 95% figure in the headline card instead |
| Blank cards on the SKU Detail page | Every card there is a single-SKU figure. Set the slicer to Single select, save a SKU as the opening selection, and let the title measure explain what to do when nothing is picked |
| Status colours on the donut | Format → Data colours → fx → Field value → `Status List Colour`. Theme order alone puts the wrong colour on each status |

---

## Build Checklist

- [ ] Query locale set to English (United Kingdom)
- [ ] No *Advanced Editor* edits in any query; exactly one *Custom Column* (`WeekStart`)
- [ ] `WeekStart` verified Monday-only with *Name of Day*
- [ ] `Zero_Week_Seed` present and appended - the 27 Dec 2010 week survives the pivot
- [ ] Sheet overlap removed - 1,044,848 rows after append
- [ ] All 11 row counts match [`../../docs/data_quality.md`](../../docs/data_quality.md)
- [ ] Fact grid = 196,300 rows, 3,775 SKUs × 52 weeks
- [ ] 100,267 zero-demand rows present (51%)
- [ ] 17 calculated columns created in dependency order
- [ ] `Dim_Week` marked as date table
- [ ] Relationships single-direction; parameter tables disconnected
- [ ] `Service Level Curve` derived from the parameter column, not typed literals
- [ ] All portfolio totals use `SUMX` over `VALUES(StockCode)`
- [ ] Theme applied before visual formatting
- [ ] Page 5 opens with a SKU selected; slicer set to Single select
- [ ] Page 4 action filter applied as a visual-level filter `= 1`
- [ ] Page 2 scatter X axis set to logarithmic and verified on screen
- [ ] Page 6 reconciliation verdict reads MATCH ON POLICY at default parameters
- [ ] Every measure placed on at least one visual - an uncompiled measure is an unvalidated measure
