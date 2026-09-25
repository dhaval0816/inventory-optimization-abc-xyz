# Power Query Transformation Guide - Ribbon Clicks Only

**Raw source → model-ready demand grid, built by hand in the Power Query ribbon. One typed formula in the whole pipeline; every other step is a menu command.**

Every step below is a menu selection in the Power Query Editor. Follow them in order and you will reproduce the exact row counts published in [`../docs/data_quality.md`](../docs/data_quality.md). The same instructions work in Excel (Data → Get Data) and in Power BI Desktop (Home → Get data) - the ribbon is the same editor in both.

> **One typed formula, and it is Step 5.1.** The `WeekStart` column is entered in the *Custom Column* dialog as `Date.StartOfWeek([InvoiceDate], Day.Monday)`. Everything else in this guide is a menu command. No *Advanced Editor*, no formula-bar edits. Power Query records each click as a step (it stores them as M internally - that is simply how the tool saves your work), but every one of those steps was created from a menu. The Applied Steps pane in §Applied Steps as documentation is the evidence.

> **Why build it this way?** A ribbon-click pipeline can be rebuilt, audited and maintained by any analyst with Excel or Power BI - the realistic audience for an inventory model in an operations team. Anything the ribbon cannot do (the running total behind ABC, the simulated inputs) is deliberately pushed into DAX calculated columns instead of being hand-coded in M. The full reasoning is in [`../powerbi/build_notes/00_BUILD_WITHOUT_M_CODE.md`](../powerbi/build_notes/00_BUILD_WITHOUT_M_CODE.md).

## Step 0 - Set the query locale (once, before anything else)

| # | Click | Why |
|---|---|---|
| 0a | Power BI: File → Options and settings → Options → Current File → Regional Settings · Excel: Data → Get Data → Query Options → Current Workbook → Regional Settings | |
| 0b | Locale for import: English (United Kingdom) → OK | The source is a UK retailer; `InvoiceDate` must be read as dd/mm, not mm/dd |

The locale controls how dates are *parsed*. It does **not** control which day a week starts on - see Step 5.1.

---

**Conventions:** `Ribbon Tab → Group → Command` · rename every query as instructed, because later steps reference queries by name · rename steps in the Applied Steps pane to the bold names given here, so the query reads as documentation.

---

## Step 1 - Import

**Goal:** both worksheets of the source workbook, appended into one query, overlap removed.

### 1.1 Connect

| # | Click | Note |
|---|---|---|
| 1 | Home → Get Data → Excel Workbook (Power BI) or Data → Get Data → From File → From Workbook (Excel) | |
| 2 | Select `data/raw/online_retail_II.xlsx` | Do not open the file in Excel first |
| 3 | In the Navigator, tick both `Year 2009-2010` and `Year 2010-2011` | Tick the sheets, not the tables |
| 4 | Click Transform Data | Not *Load*. Nothing loads until cleaning is done |

### 1.2 Promote headers on each sheet

For each of the two queries:

| # | Click |
|---|---|
| 5 | Home → Transform → Use First Row as Headers |
| 6 | Confirm the 8 column names: `Invoice`, `StockCode`, `Description`, `Quantity`, `InvoiceDate`, `Price`, `Customer ID`, `Country` |

### 1.3 Append

| # | Click | Result |
|---|---|---|
| 7 | Select `Year 2009-2010` in the Queries pane | |
| 8 | Home → Combine → Append Queries → Append Queries as New | |
| 9 | Table to append: `Year 2010-2011` → OK | |
| 10 | Rename the new query to `Sales_Raw` (right-click → Rename) | 1,067,371 rows |
| 11 | Right-click each of the two source queries → untick Enable load | They become staging queries; only cleaned output loads |

### 1.4 Remove the sheet overlap

The two worksheets both contain 1–9 December 2010. Appending them double-counts nine days of demand, which inflates average weekly demand and corrupts every safety stock figure downstream.

Remove the overlap on the first sheet's staging query, so `Sales_Raw` (which is built from it) picks the change up automatically.

| # | Click | Result |
|---|---|---|
| 12 | In the Queries pane, select the staging query `Year 2009-2010` | |
| 13 | `InvoiceDate` filter arrow → Date/Time Filters → Before… → `01/12/2010 00:00:00` → OK | The 1–9 Dec 2010 rows leave this sheet only; the `Year 2010-2011` sheet keeps them |
| 13b | Rename this step `Removed sheet overlap`, then select `Sales_Raw` again | `Sales_Raw` now reads 1,044,848 rows - 22,523 duplicated rows removed |

An explicit, named step is visible in the Applied Steps pane, so a reviewer can see the duplication was known and handled - rather than removed as a side effect of a later date filter.

---

## Step 2 - Data Typing

**Set types early.** Power Query applies filters against the current column type, and a numeric filter on a text column silently returns nothing.

| # | Column | Click | Type |
|---|---|---|---|
| 14 | `Invoice` | Click the type icon (`ABC`) in the header → Text | Text |
| 15 | `StockCode` | type icon → Text | Text |
| 16 | `Description` | type icon → Text | Text |
| 17 | `Quantity` | type icon → Whole Number | Whole Number |
| 18 | `InvoiceDate` | type icon → Date/Time | Date/Time |
| 19 | `Price` | type icon → Decimal Number | Decimal Number |
| 20 | `Customer ID` | type icon → Whole Number | Whole Number |
| 21 | `Country` | type icon → Text | Text |

Or set all eight at once: select all columns → Transform → Any Column → Detect Data Type, then correct anything it guessed wrong (it typically types `Invoice` as whole number, which breaks the cancellation filter in Step 3 - set it back to Text).

| # | Click | Why |
|---|---|---|
| 22 | Home → Reduce Rows → Remove Columns: remove `Customer ID` and `Country` | Not used in this project. Removing them early cuts refresh time and memory on 1M rows |

Rename this step: `Set data types`.

---

## Step 3 - Cleaning

Four filters. Record the row count after each one - those counts are the evidence in [`../docs/data_quality.md`](../docs/data_quality.md), and a reviewer will ask for them.

### 3.1 Remove cancellations

Cancelled invoices carry a leading `C` and negative quantities. They are reversals, not demand.

| # | Click | Result |
|---|---|---|
| 23 | `Invoice` filter arrow → Text Filters → Does Not Begin With… | |
| 24 | Enter `C` → OK | 1,025,683 rows (−19,165) |

Rename: `Removed cancellations`.

### 3.2 Remove non-positive quantities

| # | Click | Result |
|---|---|---|
| 25 | `Quantity` filter arrow → Number Filters → Greater Than… | |
| 26 | Enter `0` → OK | 1,022,290 rows (−3,393) |

Rename: `Removed zero and negative quantity`. These are returns, damages and stock adjustments - real events, but not customer demand, and including them would net down the demand history the safety stock is calculated from.

### 3.3 Remove non-positive prices

| # | Click | Result |
|---|---|---|
| 27 | `Price` filter arrow → Number Filters → Greater Than… | |
| 28 | Enter `0` → OK | 1,019,654 rows (−2,636) |

Rename: `Removed zero and negative price`. Zero-price rows are samples, giveaways and data-entry errors; they would drag the median price down and therefore understate the simulated unit cost.

### 3.4 Keep real products only - without writing a formula

The source mixes products with service codes: `POST` (postage), `M` (manual), `D` (discount), `DOT`, `BANK CHARGES`, `AMAZONFEE`, `C2`. Real product codes begin with 5 digits (`85123A`, `21257`, `10002`). The ribbon-only way to test "the first five characters are numeric" is to try the conversion and throw away the errors.

| # | Click | Why |
|---|---|---|
| 29 | Select `StockCode` → Transform → Text Column → Format → Trim | `" 85123A"` → `"85123A"` |
| 30 | Select `StockCode` → Transform → Text Column → Format → UPPERCASE | `"85123a"` and `"85123A"` become one SKU. Skipping this splits a SKU's demand history in two and halves its calculated average |
| 31 | Select `StockCode` → Add Column → From Text → Extract → First Characters → count 5 | New column `First Characters` |
| 32 | Click the new column's type icon → Whole Number | Non-numeric prefixes become `Error` cells |
| 33 | Select the new column → Home → Reduce Rows → Remove Rows → Remove Errors | Every non-product row disappears. 1,014,945 rows |
| 34 | Right-click `First Characters` → Remove | Helper column gone; the filter it performed stays |

Rename: `Kept 5-digit product codes`. This is the technique that replaces a typed `try … otherwise null` expression - Remove Errors is a ribbon button doing the work of an error-handling formula.

### 3.5 Remove the one invoice that distorts everything

| # | Click | Why |
|---|---|---|
| 35 | `Invoice` filter arrow → untick / Text Filters → Does Not Equal `541431` | 1,014,944 rows |

Invoice `541431` is a single line of 74,215 units, fully reversed nine minutes later by credit note `C541433`. The credit note was already removed in step 3.1, so leaving the original in place strands an enormous phantom sale in one week - enough on its own to distort that SKU's mean and σ and therefore its safety stock. Documented in [`../docs/data_quality.md`](../docs/data_quality.md).

Rename: `Removed reversed bulk invoice 541431`.

---

## Step 4 - Line Revenue

| # | Click | Result |
|---|---|---|
| 36 | Select `Quantity`, then Ctrl-click `Price` (order matters) | |
| 37 | Add Column → From Number → Standard → Multiply | New column `Multiplication` |
| 38 | Double-click the header → rename to `LineRevenue` | |

No formula typed - Standard → Multiply is the ribbon equivalent of `[Quantity] * [Price]`.

---

## Step 5 - Week Creation

### 5.1 Add the week column - the one typed formula, and why it has to be typed

| # | Click | Result |
|---|---|---|
| 39 | Add Column → General → Custom Column | Dialog opens |
| 40 | New column name: `WeekStart` · Formula: `Date.StartOfWeek([InvoiceDate], Day.Monday)` → OK | New column `WeekStart` |
| 41 | Right-click `WeekStart` → Change Type → Date | Date type |
| 42 | Verify: select `WeekStart` → Add Column → Date → Day → Name of Day, tick View → Column distribution and profile on the *entire data set* - `Day Name` must show one value, Monday → then delete the helper column | Monday-start weeks, proven |

**Why not the ribbon button.** *Add Column → Date → Week → Start of Week* looks like the obvious command, and it is wrong for this dataset. It calls `Date.StartOfWeek` with the first-day argument omitted, and that argument defaults to **Sunday** - the query locale does not change it ([Microsoft's `Date.StartOfWeek` reference](https://learn.microsoft.com/powerquery-m/date-startofweek) shows the default in its first example).

What that costs here is not cosmetic. With Sunday weeks the 52-week window 06 Dec 2010 → 28 Nov 2011 collapses to 51 weeks, the week of 6–12 Dec 2010 disappears entirely, and every mean, standard deviation, safety stock, reorder point and ABC class in the model moves. One dialog is the correct fix. Claiming the locale solved it would be a false claim in a portfolio piece, which is worse than typing one formula.

> **If `Day Name` shows anything but Monday**, the `Day.Monday` argument is missing from the formula. Fix it in the same dialog - do not proceed to 5.2 until the column shows Monday only; the 500,376 row count in 5.2 is the second check.

### 5.2 Keep exactly 52 complete weeks

| # | Click | Result |
|---|---|---|
| 43 | `WeekStart` filter arrow → Date Filters → Between… | |
| 44 | is after or equal to 06/12/2010, is before or equal to 28/11/2011 → OK | 500,376 rows |

Rename: `Kept 52 complete weeks`.

**Why this window.** 52 complete Monday-start weeks give exactly one seasonal cycle with no partial week at either end. The source runs to 09 Dec 2011, but that final stub is a partial week - including it would understate the demand of every SKU that sells in it and pull down σ. A partial period at the end of a demand history is the single most common cause of an understated safety stock figure.

### 5.3 Group to SKU × week

| # | Click | Result |
|---|---|---|
| 45 | Transform → Table → Group By → switch to Advanced | |
| 46 | Group by: `StockCode`, then Add grouping → `WeekStart` | |
| 47 | New column `Units` - Operation Sum, Column `Quantity` | |
| 48 | Add aggregation → `Revenue` - Operation Sum, Column `LineRevenue` | |
| 49 | OK | 96,033 rows |
| 50 | Rename the query to `Weekly_Sold` | |

96,033 rows, from 3,775 distinct SKUs. If the grid were complete it would be 196,300 - which is the entire point of the next step.

---

## Step 6 - Unpivoting (the zero-fill)

**This step matters most.** A SKU that sold nothing in week 14 currently has *no row* for week 14. Calculating σ on the rows that exist measures the variability of *weeks with sales*, not the variability of *demand*, and understates safety stock on exactly the intermittent items that most need a buffer.

The pivot → replace → unpivot round trip creates the missing rows as zeros using three ribbon commands and no join.

### 6.0 Seed the one week in which nothing sold at all

One week in this dataset - **27 Dec 2010 to 2 Jan 2011** - contains no transactions whatsoever, because the retailer closed over Christmas. That breaks the pivot trick: *Pivot Column* creates one column per value that exists, so a week with no rows anywhere produces no column, and the grid silently comes out 51 weeks wide instead of 52. Every SKU's standard deviation would then be computed over 51 observations instead of 52, including the zero.

The ribbon-only fix is a one-row seed table:

| # | Click | Result |
|---|---|---|
| 50a | Home → New Query → Enter Data | Create Table dialog |
| 50b | Three columns `StockCode`, `WeekStart`, `Units` · one row: `10002`, `2010-12-27`, `0` · name it `Zero_Week_Seed` → OK | 1-row query |
| 50c | Set types: `StockCode` Text, `WeekStart` Date, `Units` Whole Number · right-click the query → untick Enable load | Staging query |

`Units` is zero, so the seed adds nothing to any total; it exists only so the calendar stays 52 weeks wide. It is listed in [`../docs/assumptions.md`](../docs/assumptions.md) as a modelling device, not as data.

| # | Click | Result |
|---|---|---|
| 51 | Right-click `Weekly_Sold` → Reference. Rename the new query `Fact_WeeklyDemand` | Keeps `Weekly_Sold` intact for the revenue merge |
| 52 | Select `Revenue` → right-click → Remove | Pivot one measure at a time |
| 52b | Append the zero-week seed - see §6.0 below | 96,034 rows |
| 53 | Select `WeekStart` → Transform → Any Column → Pivot Column | |
| 54 | Values Column: `Units` · Advanced options → Aggregate Value Function → Don't Aggregate → OK | 3,775 rows × 52 week columns. Every gap is a `null` |
| 55 | Select all columns (click `StockCode`, then Ctrl-A in the grid) | `StockCode` holds no nulls, so including it is harmless |
| 56 | Home → Transform → Replace Values → Value to Find `null`, Replace With `0` → OK | Every gap is now a real zero |
| 57 | Select `StockCode` → right-click → Unpivot Other Columns | Do this *after* the null fill: unpivot drops null values, which would undo the whole point |
| 58 | Rename `Attribute` → `WeekStart`, `Value` → `Units` | |
| 59 | Set `WeekStart` to Date, `Units` to Whole Number | 196,300 rows |

> **Use *Don't Aggregate*, not the default Sum.** With Sum, Power Query silently collapses any duplicate `StockCode` + `WeekStart` pair. Since Step 5.3 already grouped to that exact grain, duplicates should be impossible - and if any exist, *Don't Aggregate* throws a visible error rather than hiding a data defect behind a plausible number.

### 6.1 Bring revenue back

| # | Click |
|---|---|
| 60 | Home → Combine → Merge Queries → merge `Fact_WeeklyDemand` with `Weekly_Sold` |
| 61 | Select `StockCode` and `WeekStart` in both (Ctrl-click for the second key) · Join Kind Left Outer → OK |
| 62 | Expand the new column (⇄ icon) → tick `Revenue` only · untick *Use original column name as prefix* |
| 63 | Select `Revenue` → Transform → Replace Values → `null` → `0` |

---

## Step 7 - Null Handling

Nulls carry different meanings in different columns, so each gets its own treatment. Blanket "remove all nulls" is a data-destroying habit - here it would delete the 100,267 zero-demand weeks that are the whole point of Step 6.

| Column | Null means | Treatment | Ribbon path |
|---|---|---|---|
| `Units` (post-pivot) | No sale that week | Replace with `0` - this is real information | Transform → Replace Values |
| `Revenue` (post-merge) | No sale that week | Replace with `0` | Transform → Replace Values |
| `Description` | Missing product name | Keep the row. Description feeds no calculation; dropping the row would delete real demand | Home → Keep Rows (no action) |
| `Customer ID` | Guest / unlinked order | Column removed in Step 2 | - |
| `Quantity`, `Price` | Should never be null | Remove the row if present, and record the count | Home → Remove Rows → Remove Blank Rows |
| `StockCode` | Should never be null | Remove the row and investigate before proceeding | Home → Remove Rows → Remove Blank Rows |
| `InvoiceDate` | Should never be null | Remove the row; a transaction with no date cannot be assigned to a week | Home → Remove Rows → Remove Blank Rows |

**To confirm nulls are gone** on any column: View → Data Preview → Column Quality. The *Empty* figure must read `0%` on `StockCode`, `WeekStart` and `Units` before loading.

**Do not use *Remove Blank Rows* on the post-pivot grid.** Power Query treats a row of zeros as populated, so it is safe - but a `Fill Down` or `Remove Empty` applied carelessly at this stage will eat the zero-demand weeks and quietly restore the exact bug Step 6 was built to fix.

---

## Step 8 - SKU Attributes

Build the SKU dimension. Keep every step ribbon-only; anything derived (unit cost, lead time, on-hand, ABC, XYZ) is computed later as a DAX calculated column, not here.

### 8.1 Most frequent description, without a formula

| # | Click |
|---|---|
| 64 | Right-click `Sales_Raw` → Reference → rename `SKU_Description` (load off) |
| 65 | Transform → Group By (Advanced): group by `StockCode` and `Description`; new column `n`, Operation Count Rows |
| 66 | `StockCode` filter arrow → Sort Ascending, then `n` filter arrow → Sort Descending (in that order - the second sort becomes the tie-break) |
| 67 | Select `StockCode` → right-click → Remove Duplicates | Keeps the first row per SKU = the most frequent description |
| 68 | The `n` column can stay; only `Description` is expanded in 8.2 |

> **Fallback** if the sort is not respected after a refresh (Power Query does not guarantee sort stability across a Remove Duplicates): Group By `StockCode` → Operation Max, Column `Description`. Description drives no calculation, so "most frequent" and "alphabetically last" are equally acceptable - but say which one you used.

### 8.2 Median selling price

| # | Click |
|---|---|
| 69 | `SKU_Attributes`: Source = `Sales_Raw` → Transform → Group By → group by `StockCode`, new column `MedianPrice`, Operation Median, Column `Price` |
| 70 | Home → Merge Queries with `SKU_Description` on `StockCode`, Left Outer → expand `Description` only, untick *Use original column name as prefix* |
| 70b | `Dim_SKU`: Source = `SKU_Attributes` → Merge Queries with `Excel_SKU_Policy` on `StockCode`, Left Outer → expand the seven `_Excel` columns |

Median, not average - a single wholesale line at a discounted price would drag a mean well below the item's normal selling price, and unit cost is derived from this figure.

### 8.3 Everything else moves to DAX

`UnitCost`, `LeadTimeWeeks`, `OnHand`, `Category`, `AnnualUnits`, `AvgWeekly`, `StdWeekly`, `CV`, `AnnualConsumptionValue`, `CumValueShare`, `ValueRank`, `ABC`, `XYZ`, `ABC_XYZ`, `In Excel Scope` - all 17 are DAX calculated columns on `Dim_SKU`, specified in [`../powerbi/build_notes/02_model_and_dax.md`](../powerbi/build_notes/02_model_and_dax.md).

Why these are DAX columns and not Power Query steps:

1. **There is no ribbon button for a running total.** ABC needs a cumulative value share. In Power Query that means writing M by hand (and buffering it, or the refresh re-reads the source once per row). In DAX it is one calculated column, evaluated once at refresh.
2. **Rounding has to match Excel.** The Power Query ribbon's *Rounding → Round* uses banker's rounding (round-half-to-even); Excel's `ROUND` rounds half away from zero. Rounding the simulated on-hand in Power Query would disagree with the workbook on every exact-half value. DAX `ROUND` matches Excel, so the two models agree.

---

## Step 9 - Excel Scope Export

For the Excel workbook only - the top 500 SKUs by annual revenue, 67.5% of portfolio revenue.

| # | Click | Result |
|---|---|---|
| 71 | Duplicate `Fact_WeeklyDemand` → rename `Excel_Top500_Wide` | |
| 72 | Reference `Weekly_Sold` → Group By `StockCode`, Sum `Revenue` → sort Descending → Home → Keep Rows → Keep Top Rows → 500 → Merge into `Excel_Top500_Wide` on `StockCode` with Join Kind Inner | 26,000 rows (500 × 52) |
| 73 | Transform → Pivot Column on `WeekStart`, Values `Units`, Don't Aggregate | 500 rows × 52 week columns |
| 74 | Merge in `Description`, `Category`, `Unit Cost`, `Lead Time (weeks)`, `On Hand` from `Dim_SKU` | |
| 75 | Reorder columns to: `SKU`, `Description`, `Category`, `Unit Cost`, `Lead Time (weeks)`, `On Hand`, `W1`…`W52` | Order is not cosmetic - the workbook's formulas reference by position |
| 76 | Home → Close & Load To… → Table → save as `data/clean/weekly_demand_wide.xlsx` | |

---

## Step 10 - Validation

Run every check before loading. A transformation that is not counted is not verified.

### 10.1 Row-count audit

Open View → Applied Steps and read the row count in the status bar after each step. These must match exactly:

| After step | Expected rows | Delta |
|---|---|---|
| Append both sheets | 1,067,371 | - |
| Remove sheet overlap (1–9 Dec 2010) | 1,044,848 | −22,523 |
| Remove cancellations | 1,025,683 | −19,165 |
| Remove `Quantity` ≤ 0 | 1,022,290 | −3,393 |
| Remove `Price` ≤ 0 | 1,019,654 | −2,636 |
| Keep 5-digit product codes | 1,014,945 | −4,709 |
| Remove invoice 541431 | 1,014,944 | −1 |
| Keep 52 complete weeks | 500,376 | −514,568 |
| Group to SKU × week | 96,033 | - |
| Append the zero-week seed | 96,034 | +1 |
| Zero-filled grid | 196,300 | +100,266 |
| Excel top-500 scope | 26,000 | - |

A mismatch at any line means a step was applied out of order or a filter used the wrong comparison. Fix it there - do not carry it forward.

### 10.2 Grid integrity

| # | Check | Ribbon path | Expected |
|---|---|---|---|
| 1 | Distinct SKUs | Select `StockCode` → Transform → Statistics → Count Distinct Values | 3,775 |
| 2 | Distinct weeks | Select `WeekStart` → Transform → Statistics → Count Distinct Values | 52 |
| 3 | Total rows = 3,775 × 52 | Transform → Table → Count Rows | 196,300 |
| 4 | Every SKU has 52 rows | Group By `StockCode`, Count Rows → check Min and Max | both 52 |

> Run each check on a duplicate of the query - `Transform → Statistics` and `Count Rows` replace the table with a single value. Delete the duplicate afterwards.

### 10.3 Column quality and profile

**View → Data Preview** → tick Column quality, Column distribution, Column profile, and set the profiler to Column profiling based on entire data set (bottom status bar - it defaults to the first 1,000 rows, which on 196,300 rows tells you nothing).

| Column | Expected |
|---|---|
| `StockCode` | 0% empty · 0% error · 3,775 distinct |
| `WeekStart` | 0% empty · 0% error · 52 distinct |
| `Units` | 0% empty · 0% error · minimum 0, no negatives |
| `Revenue` | 0% empty · minimum 0 |

### 10.4 Zero-fill proof

The most important check:

| Metric | Expected | Meaning |
|---|---|---|
| Rows with `Units` > 0 | 96,033 | Real transactions |
| Rows with `Units` = 0 | 100,267 | Manufactured zero-demand weeks |
| Zero share | 51% | Over half of all SKU-weeks have no sale |

Filter `Units` equals 0 and read the count. If it returns 0 rows, the pivot/unpivot round trip did not take and every safety stock figure downstream will be understated.

### 10.5 Spot check against the source

Pick three SKUs - one high-volume steady, one mid, one erratic (the workbook uses `20725`, `22795`, `21257`). For each:

1. Filter the raw source to that `StockCode` and one `WeekStart`.
2. Sum `Quantity` by hand.
3. Compare against the `Units` value in `Fact_WeeklyDemand` for the same SKU-week.

They must match exactly. Record the comparison in [`../docs/validation.md`](../docs/validation.md).

### 10.6 Before you click Close & Load

- [ ] Every query renamed (`Sales_Raw`, `Weekly_Sold`, `Zero_Week_Seed`, `Fact_WeeklyDemand`, `SKU_Description`, `SKU_Attributes`, `Dim_SKU`, `Dim_Week`, `Excel_Top500_Wide`)
- [ ] Every Applied Step renamed to something a reader understands
- [ ] Staging queries set to Enable load = off
- [ ] All 11 row counts in §10.1 confirmed
- [ ] Grid = 196,300 rows, 3,775 SKUs, 52 weeks each
- [ ] Zero rows = 100,267 confirmed present
- [ ] Data types set on every loaded column
- [ ] `Customer ID` and `Country` removed

---

## Applied Steps as documentation

When finished, `Fact_WeeklyDemand`'s Applied Steps pane should read as a plain-English audit trail:

```
Source
Promoted Headers
Appended both sheets        ← overlap already removed in the Year 2009-2010 staging query
Set data types
Removed unused columns
Removed cancellations
Removed zero and negative quantity
Removed zero and negative price
Trimmed and uppercased StockCode
Kept 5-digit product codes
Removed reversed bulk invoice 541431
Added LineRevenue
Added WeekStart
Kept 52 complete weeks
Grouped to SKU and week
Pivoted weeks
Replaced null with zero
Unpivoted weeks
Merged revenue
Set final data types
```

**Twenty-odd steps, and one typed formula that is named where it appears.** Anyone opening this query can see exactly what was done to the data and why - which is the entire argument for doing it in the ribbon rather than in code.
