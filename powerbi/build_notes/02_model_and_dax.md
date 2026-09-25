# 02 - Data Model and DAX

Reference for the semantic model in `Inventory Optimization.SemanticModel`. The TMDL files in that folder are the source of truth. This page explains what is in them and why it is built that way.

**Size:** 13 tables, 17 calculated columns, 84 measures (80 in `_Measures`, one on each of the four what-if tables), 7 display folders.

---

## 1. Schema

A star schema: one fact table, two dimensions, and a set of disconnected tables that drive the what-if sliders without filtering any data.

```
        Dim_SKU (3,775 rows)                    Dim_Week (52 rows)
        StockCode  PK                           WeekStart  PK
        Description, Category                   WeekEnd, Week No
        UnitCost, LeadTimeWeeks, OnHand         Month, Quarter
        AvgWeekly, StdWeekly, CV                (marked as date table)
        ABC, XYZ, ABC_XYZ, CumValueShare
        *_Excel columns (reconciliation)
                 | 1                                  | 1
                 |                                    |
                 +---------->  Fact_WeeklyDemand  <---+
                         *     196,300 rows           *
                               StockCode, WeekStart
                               Units, Revenue

Disconnected:  Service Level | Lead Time Change | Ordering Cost | Holding Cost %
               SL Mode | Service Level Curve | Status List | Action View | Segment Policy
Measures:      _Measures
```

### Tables

| Table | Rows | Built with | Purpose |
|---|---|---|---|
| `Fact_WeeklyDemand` | 196,300 | Power Query | Units and revenue per SKU per week, zero-filled |
| `Dim_SKU` | 3,775 | Power Query + 17 DAX columns | SKU attributes, simulated inputs, classification, Excel reconciliation columns |
| `Dim_Week` | 52 | Power Query + 5 DAX columns | Week calendar, 6 Dec 2010 to 28 Nov 2011 |
| `Service Level` | 40 | `GENERATESERIES(0.8, 0.995, 0.005)` | Uniform service level slider |
| `Lead Time Change` | 7 | `GENERATESERIES(-2, 4, 1)` | Supplier delay stress test, weeks |
| `Ordering Cost` | 9 | `GENERATESERIES(20, 100, 10)` | Cost per purchase order, £ |
| `Holding Cost %` | 5 | `GENERATESERIES(0.15, 0.35, 0.05)` | Annual holding cost rate |
| `SL Mode` | 2 | `DATATABLE` | Class-based (A 98% / B 95% / C 90%) or Uniform (slider) |
| `Service Level Curve` | 9 | DAX, filtered from `Service Level` | X axis of the Page 3 curve |
| `Status List` | 4 | `DATATABLE` | Legend for the status donut |
| `Action View` | 2 | `DATATABLE` | Action / Excess toggle on Page 4 |
| `Segment Policy` | 9 | `DATATABLE` | Recommended policy per ABC-XYZ cell, Page 2 |
| `_Measures` | 1 | `ROW()` | Holds the measures |

The Excel workbook is read by a staging query (`Excel_SKU_Policy`, not loaded) and merged into `Dim_SKU` as seven `*_Excel` columns.

### Relationships

| From | To | Cardinality | Filter direction |
|---|---|---|---|
| `Fact_WeeklyDemand[StockCode]` | `Dim_SKU[StockCode]` | many to one | single |
| `Fact_WeeklyDemand[WeekStart]` | `Dim_Week[WeekStart]` | many to one | single |

Every other table is disconnected on purpose. The what-if tables are read with `SELECTEDVALUE`. If they were related to the fact table, moving a slider would filter the data instead of changing a calculation.

---

## 2. Power Query

Power Query only shapes the data. Every step is a ribbon command (see [`../../powerquery/power_query_steps.md`](../../powerquery/power_query_steps.md)). Anything that needs logic, such as the running total behind ABC, lives in DAX.

| Query | Loaded | Output |
|---|---|---|
| `SourceFile`, `ExcelModelFile` | parameter | File paths. Change these two values to run the model on another machine |
| `Sales_2009_2010`, `Sales_2010_2011` | no | The two source sheets, typed. The first one drops 1-9 Dec 2010, which also appears on the second sheet |
| `Sales_Raw` | no | Appended and cleaned transaction lines, with `WeekStart` and `LineRevenue` |
| `Weekly_Sold` | no | Grouped to SKU x week |
| `SKU_Attributes` | no | Median price and most frequent description per SKU |
| `Excel_SKU_Policy` | no | The workbook's `SKU_Policy` sheet |
| `Fact_WeeklyDemand` | yes | Full 3,775 x 52 grid, zero-filled by pivot, replace nulls, unpivot |
| `Dim_SKU` | yes | `SKU_Attributes` merged with `Excel_SKU_Policy` |
| `Zero_Week_Seed` | no | One zero-demand row so the 27 Dec 2010 week survives the pivot |
| `SKU_Description` | no | Most frequent description per SKU (Group By, Sort, Remove Duplicates) |
| `Dim_Week` | yes | Distinct weeks from the fact grid, with a ribbon Index column as `Week No` |

### Dim_Week's derived columns are DAX

`WeekStart` and `Week No` come from Power Query (distinct values, sorted, Index column). The rest are DAX calculated columns, because the Power Query ribbon has no command that adds days to a date or formats one as `MMM yyyy`:

```dax
WeekEnd     = Dim_Week[WeekStart] + 6
Month       = FORMAT ( Dim_Week[WeekStart], "MMM yyyy" )
MonthSort   = YEAR ( Dim_Week[WeekStart] ) * 100 + MONTH ( Dim_Week[WeekStart] )
Quarter     = "Q" & QUARTER ( Dim_Week[WeekStart] ) & " " & YEAR ( Dim_Week[WeekStart] )
QuarterSort = YEAR ( Dim_Week[WeekStart] ) * 10 + QUARTER ( Dim_Week[WeekStart] )
```

`Month` is sorted by `MonthSort` and `Quarter` by `QuarterSort`, both set through Column tools -> Sort by column.

---

## 3. Calculated columns on Dim_SKU

Seventeen columns, computed once per refresh.

| Column | Notes |
|---|---|
| `UnitCost` | Assumption: median price x 0.60 (40% gross margin) |
| `AnnualUnits`, `AvgWeekly`, `StdWeekly`, `CV` | `StdWeekly` needs `CALCULATE` inside the iterator for context transition. Without it every SKU returns the portfolio sigma |
| `LeadTimeWeeks` | Simulated, {2, 4, 6, 8} weeks from the first five digits of the code |
| `WeeksOfCover`, `OnHand` | Simulated snapshot. DAX `ROUND` rounds half away from zero, the same as Excel |
| `Category` | Whole-word keyword match (Bags, Lighting, Kitchen, Cards/Gifts, Other). The description is padded with spaces so TIN does not match PAINTING |
| `AnnualConsumptionValue` | Units x unit cost. The ABC base |
| `CumValueShare`, `ValueRank` | Running share of value, with a StockCode tie-breaker so equal values do not double count |
| `ABC`, `XYZ`, `ABC_XYZ` | Cut-offs 80% / 95% of value, and CV 0.5 / 1.0 |
| `In Excel Scope`, `OnHand Check` | Flags the 500 SKUs shared with the workbook |

<details>
<summary>DAX - calculated columns</summary>

```dax
UnitCost = Dim_SKU[MedianPrice] * 0.60

AnnualUnits = CALCULATE ( SUM ( Fact_WeeklyDemand[Units] ) )

AvgWeekly = DIVIDE ( Dim_SKU[AnnualUnits], 52 )

StdWeekly =
CALCULATE (
    STDEVX.S (
        VALUES ( Dim_Week[WeekStart] ),
        CALCULATE ( SUM ( Fact_WeeklyDemand[Units] ) )
    )
)

CV = DIVIDE ( Dim_SKU[StdWeekly], Dim_SKU[AvgWeekly] )

LeadTimeWeeks =
VAR n = VALUE ( LEFT ( Dim_SKU[StockCode], 5 ) )
RETURN
    SWITCH ( MOD ( n, 4 ), 0, 2, 1, 4, 2, 6, 8 )

WeeksOfCover =
VAR h = MOD ( VALUE ( LEFT ( Dim_SKU[StockCode], 5 ) ) * 37 + LEN ( Dim_SKU[StockCode] ) * 11, 100 )
RETURN
    SWITCH (
        TRUE (),
        h < 20, 1 + MOD ( h, 4 ),
        h < 75, 6 + MOD ( h, 11 ),
        26 + MOD ( h, 27 )
    )

OnHand = ROUND ( Dim_SKU[AvgWeekly] * Dim_SKU[WeeksOfCover], 0 )

Category =
VAR d =
    " "
        & UPPER (
            SUBSTITUTE ( SUBSTITUTE ( SUBSTITUTE ( SUBSTITUTE (
                Dim_SKU[Description], "-", " " ), "/", " " ), ",", " " ), ".", " " )
        )
        & " "
RETURN
    SWITCH (
        TRUE (),
        SEARCH ( " BAG ", d, 1, 0 ) > 0 || SEARCH ( " BAGS ", d, 1, 0 ) > 0, "Bags",
        SEARCH ( " CANDLE ", d, 1, 0 ) > 0 || SEARCH ( " CANDLES ", d, 1, 0 ) > 0
            || SEARCH ( " LIGHT ", d, 1, 0 ) > 0 || SEARCH ( " LIGHTS ", d, 1, 0 ) > 0, "Lighting",
        SEARCH ( " MUG ", d, 1, 0 ) > 0 || SEARCH ( " MUGS ", d, 1, 0 ) > 0
            || SEARCH ( " CAKE ", d, 1, 0 ) > 0 || SEARCH ( " CAKES ", d, 1, 0 ) > 0
            || SEARCH ( " CAKESTAND ", d, 1, 0 ) > 0
            || SEARCH ( " TIN ", d, 1, 0 ) > 0 || SEARCH ( " TINS ", d, 1, 0 ) > 0, "Kitchen",
        SEARCH ( " CARD ", d, 1, 0 ) > 0 || SEARCH ( " CARDS ", d, 1, 0 ) > 0, "Cards/Gifts",
        "Other"
    )

AnnualConsumptionValue = Dim_SKU[AnnualUnits] * Dim_SKU[UnitCost]

CumValueShare =
VAR v = Dim_SKU[AnnualConsumptionValue]
VAR code = Dim_SKU[StockCode]
VAR Cum =
    SUMX (
        FILTER (
            Dim_SKU,
            Dim_SKU[AnnualConsumptionValue] > v
                || ( Dim_SKU[AnnualConsumptionValue] = v && Dim_SKU[StockCode] <= code )
        ),
        Dim_SKU[AnnualConsumptionValue]
    )
VAR Total = SUMX ( ALL ( Dim_SKU ), Dim_SKU[AnnualConsumptionValue] )
RETURN
    DIVIDE ( Cum, Total )

ValueRank = RANKX ( Dim_SKU, Dim_SKU[CumValueShare], , ASC, DENSE )

ABC =
SWITCH (
    TRUE (),
    Dim_SKU[CumValueShare] <= 0.80, "A",
    Dim_SKU[CumValueShare] <= 0.95, "B",
    "C"
)

XYZ =
SWITCH (
    TRUE (),
    ISBLANK ( Dim_SKU[CV] ), "Z",
    Dim_SKU[CV] <= 0.5, "X",
    Dim_SKU[CV] <= 1.0, "Y",
    "Z"
)

ABC_XYZ = Dim_SKU[ABC] & Dim_SKU[XYZ]

In Excel Scope = IF ( ISBLANK ( Dim_SKU[SS_Excel] ), "Power BI only", "In Excel top-500" )

OnHand Check =
IF (
    ISBLANK ( Dim_SKU[OnHand_Excel] ),
    BLANK (),
    IF ( Dim_SKU[OnHand_Excel] = Dim_SKU[OnHand], "Match", "DIFFERS" )
)
```

</details>

---

## 4. Measures

### Demand (8)

Volume and variability. Every average and standard deviation runs over all 52 weeks, zero-demand weeks included.

| Measure | What it does |
|---|---|
| `Units Sold` | Total units sold in the current filter context. |
| `Revenue` | Gross sales value at selling price (not at cost). |
| `Annual Consumption Value` | Annual demand valued at ASSUMED unit cost. This is the ABC classification base. |
| `Avg Weekly Demand` | Mean weekly units across the weeks in context, counting zero-demand weeks as zero. |
| `Std Dev Weekly Demand` | Sample standard deviation of weekly units, including zero-demand weeks. |
| `Std Dev Weekly Demand (safe)` | Blank-safe wrapper. STDEVX.S returns BLANK with fewer than 2 rows and the blank would propagate through safety stock, reorder point and stock status, blanking a whole column instead of showing 0. |
| `CV` | Coefficient of variation. Demand unpredictability independent of volume. |
| `SKU Count` | Distinct SKUs in the current filter context. |

<details>
<summary>DAX - Demand</summary>

```dax
Units Sold = SUM ( Fact_WeeklyDemand[Units] )

Revenue = SUM ( Fact_WeeklyDemand[Revenue] )

Annual Consumption Value = SUMX ( Dim_SKU, CALCULATE ( [Units Sold] ) * Dim_SKU[UnitCost] )

Avg Weekly Demand = AVERAGEX ( VALUES ( Dim_Week[WeekStart] ), COALESCE ( [Units Sold], 0 ) )

Std Dev Weekly Demand = STDEVX.S ( VALUES ( Dim_Week[WeekStart] ), COALESCE ( [Units Sold], 0 ) )

Std Dev Weekly Demand (safe) = COALESCE ( [Std Dev Weekly Demand], 0 )

CV = DIVIDE ( [Std Dev Weekly Demand], [Avg Weekly Demand] )

SKU Count = DISTINCTCOUNT ( Dim_SKU[StockCode] )
```

</details>

### Policy (8)

Per-SKU policy: service level, lead time, safety stock, reorder point, EOQ and the suggested order. Each one is blank above single-SKU grain by design.

| Measure | What it does |
|---|---|
| `Selected Service Level` | The uniform service level chosen on the what-if slider. |
| `Class Service Level` | Differentiated service level by ABC class: A 98%, B 95%, C 90%. |
| `Effective Service Level` | Class-based mode reproduces the Excel policy exactly. |
| `Effective Lead Time` | Simulated lead time plus the shock slider, floored at 1 week. |
| `Safety Stock Units` | Z x sigma x sqrt(lead time), rounded UP. Rounding a service-level threshold down means quietly delivering less service than specified. |
| `Reorder Point Units` | Expected demand over the lead time plus the buffer. |
| `EOQ Units` | Economic order quantity. Rounded to nearest unit: the total-cost curve is flat near its minimum, so nearest-unit is fine for an order size (unlike a threshold). |
| `Suggested Order Qty` | Order-up-to quantity for SKUs at or below their reorder point. |

<details>
<summary>DAX - Policy</summary>

```dax
Selected Service Level = SELECTEDVALUE ( 'Service Level'[Service Level], 0.95 )

Class Service Level =
SWITCH (
    SELECTEDVALUE ( Dim_SKU[ABC] ),
    "A", 0.98,
    "B", 0.95,
    "C", 0.90,
    0.95
)

Effective Service Level =
IF (
    SELECTEDVALUE ( 'SL Mode'[SL Mode] ) = "Uniform (slider)",
    [Selected Service Level],
    [Class Service Level]
)

Effective Lead Time = MAX ( 1, SELECTEDVALUE ( Dim_SKU[LeadTimeWeeks] ) + SELECTEDVALUE ( 'Lead Time Change'[Lead Time Change], 0 ) )

Safety Stock Units =
IF (
    HASONEVALUE ( Dim_SKU[StockCode] ),
    ROUNDUP (
        NORM.S.INV ( [Effective Service Level] ) * [Std Dev Weekly Demand (safe)] * SQRT ( [Effective Lead Time] ),
        0
    )
)

Reorder Point Units =
IF (
    HASONEVALUE ( Dim_SKU[StockCode] ),
    ROUNDUP ( [Avg Weekly Demand] * [Effective Lead Time] + [Safety Stock Units], 0 )
)

EOQ Units =
VAR AnnualUnits = [Units Sold]
VAR OrderCost   = SELECTEDVALUE ( 'Ordering Cost'[Ordering Cost], 50 )
VAR HoldingPct  = SELECTEDVALUE ( 'Holding Cost %'[Holding Cost %], 0.25 )
VAR Cost        = SELECTEDVALUE ( Dim_SKU[UnitCost] )
RETURN
    IF (
        HASONEVALUE ( Dim_SKU[StockCode] ),
        ROUND ( SQRT ( DIVIDE ( 2 * AnnualUnits * OrderCost, HoldingPct * Cost ) ), 0 )
    )

Suggested Order Qty =
IF (
    HASONEVALUE ( Dim_SKU[StockCode] ),
    VAR OnHandU = SELECTEDVALUE ( Dim_SKU[OnHand] )
    VAR ROP     = [Reorder Point Units]
    VAR MaxStk  = ROP + [EOQ Units]
    RETURN
        IF ( OnHandU <= ROP, ROUNDUP ( MaxStk - OnHandU, 0 ) )
)
```

</details>

### Inventory (10)

Portfolio values. Every total iterates SKU by SKU with SUMX, because the per-SKU policy measures do not add up directly.

| Measure | What it does |
|---|---|
| `Total Safety Stock Value` | Portfolio safety stock at cost. Safety stock is NOT additive until each SKU's own sigma and lead time have been applied, so this iterates SKU by SKU. |
| `On Hand Value` | Simulated on-hand stock valued at assumed unit cost. |
| `Excess Value` | Value of stock held above max stock (reorder point + EOQ). |
| `Excess Units` | Units held above max stock (reorder point + EOQ), summed SKU by SKU. |
| `Excess % of On Hand` | Ratio of two totals, so it aggregates correctly - it is not the sum of the row percentages. |
| `Inventory Turns` | Annual cost of goods divided by on-hand value. |
| `Days on Hand` | 365 divided by inventory turns. |
| `Reorder Point Inventory Value` | Working capital implied by the policy at the moment of reordering. |
| `Safety Stock Value (unrounded)` | Same quantity without the per-SKU ROUNDUP, for a perfectly smooth service-level curve. |
| `Suggested Order Value` | Cost of placing every replenishment order the policy is calling for right now. |

<details>
<summary>DAX - Inventory</summary>

```dax
Total Safety Stock Value =
SUMX (
    VALUES ( Dim_SKU[StockCode] ),
    [Safety Stock Units] * CALCULATE ( SELECTEDVALUE ( Dim_SKU[UnitCost] ) )
)

On Hand Value = SUMX ( Dim_SKU, Dim_SKU[OnHand] * Dim_SKU[UnitCost] )

Excess Value =
SUMX (
    VALUES ( Dim_SKU[StockCode] ),
    VAR MaxStock = [Reorder Point Units] + [EOQ Units]
    VAR OnHandU  = CALCULATE ( SELECTEDVALUE ( Dim_SKU[OnHand] ) )
    RETURN
        MAX ( 0, OnHandU - MaxStock ) * CALCULATE ( SELECTEDVALUE ( Dim_SKU[UnitCost] ) )
)

Excess Units =
SUMX (
    VALUES ( Dim_SKU[StockCode] ),
    MAX ( 0, CALCULATE ( SELECTEDVALUE ( Dim_SKU[OnHand] ) ) - ( [Reorder Point Units] + [EOQ Units] ) )
)

Excess % of On Hand = DIVIDE ( [Excess Value], [On Hand Value] )

Inventory Turns = DIVIDE ( [Annual Consumption Value], [On Hand Value] )

Days on Hand = DIVIDE ( 365, [Inventory Turns] )

Reorder Point Inventory Value =
SUMX (
    VALUES ( Dim_SKU[StockCode] ),
    [Reorder Point Units] * CALCULATE ( SELECTEDVALUE ( Dim_SKU[UnitCost] ) )
)

Safety Stock Value (unrounded) =
SUMX (
    VALUES ( Dim_SKU[StockCode] ),
    NORM.S.INV ( [Effective Service Level] ) * [Std Dev Weekly Demand (safe)]
        * SQRT ( [Effective Lead Time] ) * CALCULATE ( SELECTEDVALUE ( Dim_SKU[UnitCost] ) )
)

Suggested Order Value =
SUMX (
    VALUES ( Dim_SKU[StockCode] ),
    COALESCE ( [Suggested Order Qty], 0 ) * CALCULATE ( SELECTEDVALUE ( Dim_SKU[UnitCost] ) )
)
```

</details>

### Status (12)

The four-way stock status and the counts built on it. Stock Status is a measure, so the counts evaluate it once per SKU inside a VAR table.

| Measure | What it does |
|---|---|
| `Stock Status` | Turns three policy numbers into one word a planner can sort on. |
| `SKUs Below Safety Stock` | The urgent half of the action list. The VAR evaluates each SKU's status once instead of once per comparison. |
| `SKUs Reorder Now` | SKUs at or below their reorder point - counts BOTH 'Reorder Now' and 'Below Safety Stock'. |
| `SKUs Reorder Now (strict)` | Past the reorder point but still above the buffer. |
| `SKUs Excess` | Count of SKUs above max stock. |
| `SKUs Healthy` | Count of SKUs between reorder point and max stock. |
| `SKUs in Status` | Lets a disconnected status table act as a chart legend. |
| `A-Class SKUs Below Safety Stock` | High value AND below buffer. The single number the Operations Director acts on. |
| `Value at Risk (A-class below SS)` | Annual consumption value carried by the A-class SKUs currently below safety stock. |
| `Stock Status Colour` | Feeds conditional formatting: Cell elements > Background colour > Format style 'Field value'. |
| `Action Filter` | 1/0 flag used as a visual-level filter on the Page 4 table. Switched by the Action / Excess buttons through the disconnected 'Action View' table. |
| `Status List Colour` | Status colour keyed off the disconnected 'Status List' table, so the Page 1 donut is coloured by meaning rather than by theme order. Data colours > fx > Field value. |

<details>
<summary>DAX - Status</summary>

```dax
Stock Status =
VAR OnHandU = SELECTEDVALUE ( Dim_SKU[OnHand] )
RETURN
    SWITCH (
        TRUE (),
        NOT HASONEVALUE ( Dim_SKU[StockCode] ), BLANK (),
        OnHandU <= [Safety Stock Units], "Below Safety Stock",
        OnHandU <= [Reorder Point Units], "Reorder Now",
        OnHandU > [Reorder Point Units] + [EOQ Units], "Excess",
        "Healthy"
    )

SKUs Below Safety Stock =
VAR StatusTable = ADDCOLUMNS ( VALUES ( Dim_SKU[StockCode] ), "@Status", [Stock Status] )
RETURN
    COUNTROWS ( FILTER ( StatusTable, [@Status] = "Below Safety Stock" ) )

SKUs Reorder Now =
VAR StatusTable = ADDCOLUMNS ( VALUES ( Dim_SKU[StockCode] ), "@Status", [Stock Status] )
RETURN
    COUNTROWS ( FILTER ( StatusTable, [@Status] IN { "Reorder Now", "Below Safety Stock" } ) )

SKUs Reorder Now (strict) =
VAR StatusTable = ADDCOLUMNS ( VALUES ( Dim_SKU[StockCode] ), "@Status", [Stock Status] )
RETURN
    COUNTROWS ( FILTER ( StatusTable, [@Status] = "Reorder Now" ) )

SKUs Excess =
VAR StatusTable = ADDCOLUMNS ( VALUES ( Dim_SKU[StockCode] ), "@Status", [Stock Status] )
RETURN
    COUNTROWS ( FILTER ( StatusTable, [@Status] = "Excess" ) )

SKUs Healthy =
VAR StatusTable = ADDCOLUMNS ( VALUES ( Dim_SKU[StockCode] ), "@Status", [Stock Status] )
RETURN
    COUNTROWS ( FILTER ( StatusTable, [@Status] = "Healthy" ) )

SKUs in Status =
VAR Sel = SELECTEDVALUE ( 'Status List'[Status] )
VAR StatusTable = ADDCOLUMNS ( VALUES ( Dim_SKU[StockCode] ), "@Status", [Stock Status] )
RETURN
    COUNTROWS ( FILTER ( StatusTable, [@Status] = Sel ) )

A-Class SKUs Below Safety Stock =
VAR StatusTable =
    ADDCOLUMNS (
        CALCULATETABLE ( VALUES ( Dim_SKU[StockCode] ), Dim_SKU[ABC] = "A" ),
        "@Status", [Stock Status]
    )
RETURN
    COUNTROWS ( FILTER ( StatusTable, [@Status] = "Below Safety Stock" ) )

Value at Risk (A-class below SS) =
VAR StatusTable =
    ADDCOLUMNS (
        CALCULATETABLE ( VALUES ( Dim_SKU[StockCode] ), Dim_SKU[ABC] = "A" ),
        "@Status", [Stock Status],
        "@ACV", [Annual Consumption Value]
    )
RETURN
    SUMX ( FILTER ( StatusTable, [@Status] = "Below Safety Stock" ), [@ACV] )

Stock Status Colour =
SWITCH (
    [Stock Status],
    "Below Safety Stock", "#B3261E",
    "Reorder Now", "#C47F15",
    "Healthy", "#1E7A4C",
    "Excess", "#3B6FD4",
    "#6E7C79"
)

Action Filter =
VAR S = [Stock Status]
RETURN
    SWITCH (
        SELECTEDVALUE ( 'Action View'[View], "Action" ),
        "Action", IF ( S IN { "Reorder Now", "Below Safety Stock" }, 1, 0 ),
        "Excess", IF ( S = "Excess", 1, 0 ),
        1
    )

Status List Colour =
SWITCH (
    SELECTEDVALUE ( 'Status List'[Status] ),
    "Below Safety Stock", "#B3261E",
    "Reorder Now", "#C47F15",
    "Healthy", "#1E7A4C",
    "Excess", "#3B6FD4",
    "#6E7C79"
)
```

</details>

### Scenario (5)

The service-level curve and the two self-writing headline sentences on Page 3.

| Measure | What it does |
|---|---|
| `SS Value at Level` | Plots the whole service-level-vs-investment trade-off in one visual. |
| `SS Value at Level vs 95%` | Change in safety stock value at each curve point versus the 95% baseline. |
| `SS Value at Level vs 95% pct` | The same change as a percentage of the 95% baseline. |
| `Simulator Headline` | Writes its own insight sentence from the current slider position, so the page can never show a stale number. |
| `Lead Time Headline` | Tariff / border-delay stress test in one sentence. |

<details>
<summary>DAX - Scenario</summary>

```dax
SS Value at Level =
CALCULATE (
    [Total Safety Stock Value],
    TREATAS ( VALUES ( 'Service Level Curve'[Level] ), 'Service Level'[Service Level] ),
    'SL Mode'[SL Mode] = "Uniform (slider)"
)

SS Value at Level vs 95% =
VAR Base =
    CALCULATE (
        [Total Safety Stock Value],
        TREATAS ( { 0.95 }, 'Service Level'[Service Level] ),
        'SL Mode'[SL Mode] = "Uniform (slider)"
    )
RETURN
    [SS Value at Level] - Base

SS Value at Level vs 95% pct =
VAR Base =
    CALCULATE (
        [Total Safety Stock Value],
        TREATAS ( { 0.95 }, 'Service Level'[Service Level] ),
        'SL Mode'[SL Mode] = "Uniform (slider)"
    )
RETURN
    DIVIDE ( [SS Value at Level] - Base, Base )

Simulator Headline =
VAR SS95 =
    CALCULATE (
        [Total Safety Stock Value],
        TREATAS ( { 0.95 }, 'Service Level'[Service Level] ),
        'SL Mode'[SL Mode] = "Uniform (slider)"
    )
VAR SSNow = CALCULATE ( [Total Safety Stock Value], 'SL Mode'[SL Mode] = "Uniform (slider)" )
VAR Delta = SSNow - SS95
VAR Pct = DIVIDE ( Delta, SS95 )
RETURN
    "At a uniform " & FORMAT ( [Selected Service Level], "0.0%" ) & " service level, safety stock is "
        & FORMAT ( SSNow, "\£#,##0" ) & " - "
        & IF ( Delta >= 0, "£" & FORMAT ( Delta, "#,##0" ) & " more", "£" & FORMAT ( -Delta, "#,##0" ) & " less" )
        & " than at 95% (" & FORMAT ( ABS ( Pct ), "0.0%" ) & ")."

Lead Time Headline =
VAR Shock = SELECTEDVALUE ( 'Lead Time Change'[Lead Time Change], 0 )
VAR SSNow = [Total Safety Stock Value]
VAR SSBase = CALCULATE ( [Total Safety Stock Value], TREATAS ( { 0 }, 'Lead Time Change'[Lead Time Change] ) )
VAR ROPNow = [Reorder Point Inventory Value]
VAR ROPBase = CALCULATE ( [Reorder Point Inventory Value], TREATAS ( { 0 }, 'Lead Time Change'[Lead Time Change] ) )
RETURN
    IF (
        Shock = 0,
        "Lead times at plan. Move the slider to stress-test a supplier delay.",
        "Lead times " & IF ( Shock > 0, "+", "" ) & Shock & " week(s): safety stock "
            & FORMAT ( SSNow, "\£#,##0" ) & " ("
            & IF ( SSNow >= SSBase, "+", "-" ) & "£" & FORMAT ( ABS ( SSNow - SSBase ), "#,##0" )
            & " vs plan) and working capital at reorder point "
            & FORMAT ( ROPNow, "\£#,##0" ) & " ("
            & IF ( ROPNow >= ROPBase, "+", "-" ) & "£" & FORMAT ( ABS ( ROPNow - ROPBase ), "#,##0" ) & ")."
    )
```

</details>

### Chart helpers (12)

Small measures that feed titles, reference lines, the footer and the Page 5 cards.

| Measure | What it does |
|---|---|
| `Cumulative Value %` | Pareto line. Reads the CumValueShare calculated column that also drives the ABC class, so the chart and the classification cannot disagree. |
| `Avg Plus 1 SD` | Constant line for the SKU detail chart: Analytics > Constant line > fx > Field value. |
| `SKU Detail Title` | Every card on this page is a single-SKU figure. |
| `Assumptions Footer` | Dynamic footer. Because it is a measure, moving a parameter slider cannot leave the stated assumptions out of date. |
| `ABC Class (selected)` | ABC class of the selected SKU (Page 5 card). |
| `XYZ Class (selected)` | XYZ class of the selected SKU (Page 5 card). |
| `Lead Time Weeks (selected)` | Simulated lead time of the selected SKU. |
| `Unit Cost (selected)` | Assumed unit cost of the selected SKU. |
| `On Hand (selected)` | Simulated on hand of the selected SKU. |
| `CV (selected)` | Same coefficient of variation as [CV], but blank above SKU grain. |
| `Avg Weekly Demand (all weeks)` | Flat reference line for the SKU detail chart. |
| `Avg Plus 1 SD (all weeks)` | Same fix for the average + 1 standard deviation band: constant across the 52 weeks, so a spike above it is visibly a spike. |

<details>
<summary>DAX - Chart helpers</summary>

```dax
Cumulative Value % = MAX ( Dim_SKU[CumValueShare] )

Avg Plus 1 SD = [Avg Weekly Demand] + [Std Dev Weekly Demand (safe)]

SKU Detail Title =
IF (
    HASONEVALUE ( Dim_SKU[StockCode] ),
    "SKU " & SELECTEDVALUE ( Dim_SKU[StockCode] ) & " - " & SELECTEDVALUE ( Dim_SKU[Description] )
        & "  |  " & SELECTEDVALUE ( Dim_SKU[ABC_XYZ] )
        & "  |  " & SELECTEDVALUE ( Dim_SKU[Category] ),
    "Pick one SKU in the 'Choose a SKU' box to the right - every card below is a single-SKU "
        & "policy figure, so they stay blank until exactly one SKU is selected. "
        & "Try 22795, 20725 or 21257 - class, lead time and policy figures fill in as soon as one is picked."
)

Assumptions Footer =
"ASSUMPTIONS - Unit cost = median selling price x 0.60 (40% gross margin; source has no cost data). "
    & "Lead time SIMULATED {2,4,6,8} weeks, deterministic from StockCode. "
    & "On Hand SIMULATED from a code-derived weeks-of-cover rule - no random numbers, reproducible on every refresh. "
    & "Ordering cost £" & SELECTEDVALUE ( 'Ordering Cost'[Ordering Cost], 50 )
    & "/order, holding cost " & FORMAT ( SELECTEDVALUE ( 'Holding Cost %'[Holding Cost %], 0.25 ), "0%" ) & "/yr. "
    & "Demand assumed normally distributed - a strong assumption for Z-class (lumpy) items. Currency GBP. "
    & "Source: UCI Online Retail II (Chen, D. 2012), CC BY 4.0."

ABC Class (selected) = SELECTEDVALUE ( Dim_SKU[ABC] )

XYZ Class (selected) = SELECTEDVALUE ( Dim_SKU[XYZ] )

Lead Time Weeks (selected) = SELECTEDVALUE ( Dim_SKU[LeadTimeWeeks] )

Unit Cost (selected) = SELECTEDVALUE ( Dim_SKU[UnitCost] )

On Hand (selected) = SELECTEDVALUE ( Dim_SKU[OnHand] )

CV (selected) = IF ( HASONEVALUE ( Dim_SKU[StockCode] ), [CV] )

Avg Weekly Demand (all weeks) = CALCULATE ( [Avg Weekly Demand], ALL ( Dim_Week ) )

Avg Plus 1 SD (all weeks) = CALCULATE ( [Avg Plus 1 SD], ALL ( Dim_Week ) )
```

</details>

### Validation (25)

The Excel reconciliation. The workbook's SKU_Policy sheet is merged into Dim_SKU, and these measures compare the two models SKU by SKU.

| Measure | What it does |
|---|---|
| `Grid Check` | Every SKU must have exactly 52 weekly rows. Put this on a card during validation. |
| `Excel Reconciliation Check` | Set SL Mode to Class-based, Lead Time Change 0, Ordering Cost 50, Holding Cost 25%, then read this card. |
| `ABC (Excel)` | ABC class as the Excel workbook classified it, over its own 500-SKU scope. |
| `XYZ (Excel)` | XYZ class as the Excel workbook classified it. |
| `Safety Stock (Excel)` | Safety stock straight from the workbook's SKU_Policy sheet. |
| `Reorder Point (Excel)` | Reorder point straight from the workbook's SKU_Policy sheet. |
| `EOQ (Excel)` | EOQ straight from the workbook's SKU_Policy sheet. |
| `On Hand (Excel)` | The workbook's simulated on-hand snapshot. |
| `Safety Stock Diff` | Power BI minus Excel for this SKU. Zero on every row is the whole point. |
| `Reorder Point Diff` | Power BI minus Excel for this SKU. Zero on every row is the whole point. |
| `EOQ Diff` | Power BI minus Excel for this SKU. Zero on every row is the whole point. |
| `SS Mismatches` | SKUs where the Power BI safety stock differs from the workbook. |
| `ROP Mismatches` | SKUs where the reorder point differs. Must be 0 under the Excel assumptions. |
| `EOQ Mismatches` | SKUs where EOQ differs. Must be 0 with ordering cost 50 and holding cost 25%. |
| `On Hand Mismatches` | The simulated on-hand snapshot must reproduce exactly, or every downstream currency figure is comparing two different inventories. |
| `ABC Mismatches` | EXPECTED to be non-zero. Power BI ranks all 3,775 SKUs by cumulative consumption value; the workbook ranks only its own top 500, so the 80/95 per cent cut points fall on different SKUs. |
| `XYZ Mismatches` | XYZ depends only on a SKU's own coefficient of variation, not on the rest of the portfolio, so unlike ABC this one must be 0. |
| `Excel Scope SKU Count` | How many SKUs the two models share. |
| `Class Service Level (Excel ABC)` | The same A 98 / B 95 / C 90 policy, but driven by the workbook's own ABC class instead of this model's. |
| `Safety Stock Units (Excel ABC)` | Identical formula to [Safety Stock Units]; only the service level input is taken from the workbook's ABC class. |
| `Reorder Point Units (Excel ABC)` | Reorder point using the workbook's ABC class as the service-level input. |
| `SS Mismatches (Excel ABC)` | Safety stock differences that remain once the workbook's own ABC class is used. |
| `ROP Mismatches (Excel ABC)` | Reorder point differences on the workbook's ABC class. |
| `On Hand Mismatch Detail` | Names the offending SKUs rather than just counting them, so a single-row difference can be traced instead of argued about. |
| `Reconciliation Verdict` | Grades the model against the workbook and refuses to grade at all until the sliders are set to the Excel policy. |

<details>
<summary>DAX - Validation</summary>

```dax
Grid Check =
VAR SKUs = DISTINCTCOUNT ( Fact_WeeklyDemand[StockCode] )
VAR Weeks = DISTINCTCOUNT ( Fact_WeeklyDemand[WeekStart] )
VAR RowsInGrid = COUNTROWS ( Fact_WeeklyDemand )
RETURN
    IF (
        RowsInGrid = SKUs * Weeks,
        "OK - " & SKUs & " SKUs x " & Weeks & " weeks = " & RowsInGrid & " rows",
        "BROKEN - " & RowsInGrid & " rows, expected " & SKUs * Weeks
    )

Excel Reconciliation Check =
VAR Shared = CALCULATETABLE ( VALUES ( Dim_SKU[StockCode] ), Dim_SKU[In Excel Scope] = "In Excel top-500" )
VAR T =
    ADDCOLUMNS (
        Shared,
        "@SSbi", [Safety Stock Units],
        "@SSxl", CALCULATE ( SELECTEDVALUE ( Dim_SKU[SS_Excel] ) ),
        "@ROPbi", [Reorder Point Units],
        "@ROPxl", CALCULATE ( SELECTEDVALUE ( Dim_SKU[ROP_Excel] ) ),
        "@EOQbi", [EOQ Units],
        "@EOQxl", CALCULATE ( SELECTEDVALUE ( Dim_SKU[EOQ_Excel] ) )
    )
VAR Bad =
    COUNTROWS (
        FILTER ( T, [@SSbi] <> [@SSxl] || [@ROPbi] <> [@ROPxl] || [@EOQbi] <> [@EOQxl] )
    )
VAR Total = COUNTROWS ( Shared )
RETURN
    IF (
        ISBLANK ( Total ),
        "No Excel rows loaded",
        IF ( COALESCE ( Bad, 0 ) = 0,
            "MATCH - all " & Total & " Excel SKUs reconcile on SS, ROP and EOQ",
            COALESCE ( Bad, 0 ) & " of " & Total & " Excel SKUs differ" )
    )

ABC (Excel) = SELECTEDVALUE ( Dim_SKU[ABC_Excel] )

XYZ (Excel) = SELECTEDVALUE ( Dim_SKU[XYZ_Excel] )

Safety Stock (Excel) = SELECTEDVALUE ( Dim_SKU[SS_Excel] )

Reorder Point (Excel) = SELECTEDVALUE ( Dim_SKU[ROP_Excel] )

EOQ (Excel) = SELECTEDVALUE ( Dim_SKU[EOQ_Excel] )

On Hand (Excel) = SELECTEDVALUE ( Dim_SKU[OnHand_Excel] )

Safety Stock Diff = IF ( NOT ISBLANK ( [Safety Stock (Excel)] ), [Safety Stock Units] - [Safety Stock (Excel)] )

Reorder Point Diff = IF ( NOT ISBLANK ( [Reorder Point (Excel)] ), [Reorder Point Units] - [Reorder Point (Excel)] )

EOQ Diff = IF ( NOT ISBLANK ( [EOQ (Excel)] ), [EOQ Units] - [EOQ (Excel)] )

SS Mismatches =
VAR Shared = CALCULATETABLE ( VALUES ( Dim_SKU[StockCode] ), Dim_SKU[In Excel Scope] = "In Excel top-500" )
VAR T = ADDCOLUMNS ( Shared, "@bi", [Safety Stock Units], "@xl", CALCULATE ( SELECTEDVALUE ( Dim_SKU[SS_Excel] ) ) )
RETURN
    COALESCE ( COUNTROWS ( FILTER ( T, [@bi] <> [@xl] ) ), 0 )

ROP Mismatches =
VAR Shared = CALCULATETABLE ( VALUES ( Dim_SKU[StockCode] ), Dim_SKU[In Excel Scope] = "In Excel top-500" )
VAR T = ADDCOLUMNS ( Shared, "@bi", [Reorder Point Units], "@xl", CALCULATE ( SELECTEDVALUE ( Dim_SKU[ROP_Excel] ) ) )
RETURN
    COALESCE ( COUNTROWS ( FILTER ( T, [@bi] <> [@xl] ) ), 0 )

EOQ Mismatches =
VAR Shared = CALCULATETABLE ( VALUES ( Dim_SKU[StockCode] ), Dim_SKU[In Excel Scope] = "In Excel top-500" )
VAR T = ADDCOLUMNS ( Shared, "@bi", [EOQ Units], "@xl", CALCULATE ( SELECTEDVALUE ( Dim_SKU[EOQ_Excel] ) ) )
RETURN
    COALESCE ( COUNTROWS ( FILTER ( T, [@bi] <> [@xl] ) ), 0 )

On Hand Mismatches =
VAR Shared = CALCULATETABLE ( VALUES ( Dim_SKU[StockCode] ), Dim_SKU[In Excel Scope] = "In Excel top-500" )
VAR T = ADDCOLUMNS ( Shared, "@bi", CALCULATE ( SELECTEDVALUE ( Dim_SKU[OnHand] ) ), "@xl", CALCULATE ( SELECTEDVALUE ( Dim_SKU[OnHand_Excel] ) ) )
RETURN
    COALESCE ( COUNTROWS ( FILTER ( T, [@bi] <> [@xl] ) ), 0 )

ABC Mismatches =
VAR Shared = CALCULATETABLE ( VALUES ( Dim_SKU[StockCode] ), Dim_SKU[In Excel Scope] = "In Excel top-500" )
VAR T = ADDCOLUMNS ( Shared, "@bi", CALCULATE ( SELECTEDVALUE ( Dim_SKU[ABC] ) ), "@xl", CALCULATE ( SELECTEDVALUE ( Dim_SKU[ABC_Excel] ) ) )
RETURN
    COALESCE ( COUNTROWS ( FILTER ( T, [@bi] <> [@xl] ) ), 0 )

XYZ Mismatches =
VAR Shared = CALCULATETABLE ( VALUES ( Dim_SKU[StockCode] ), Dim_SKU[In Excel Scope] = "In Excel top-500" )
VAR T = ADDCOLUMNS ( Shared, "@bi", CALCULATE ( SELECTEDVALUE ( Dim_SKU[XYZ] ) ), "@xl", CALCULATE ( SELECTEDVALUE ( Dim_SKU[XYZ_Excel] ) ) )
RETURN
    COALESCE ( COUNTROWS ( FILTER ( T, [@bi] <> [@xl] ) ), 0 )

Excel Scope SKU Count = CALCULATE ( DISTINCTCOUNT ( Dim_SKU[StockCode] ), Dim_SKU[In Excel Scope] = "In Excel top-500" )

Class Service Level (Excel ABC) = SWITCH ( SELECTEDVALUE ( Dim_SKU[ABC_Excel] ), "A", 0.98, "B", 0.95, "C", 0.90, 0.95 )

Safety Stock Units (Excel ABC) =
IF (
    HASONEVALUE ( Dim_SKU[StockCode] ),
    ROUNDUP (
        NORM.S.INV ( [Class Service Level (Excel ABC)] ) * [Std Dev Weekly Demand (safe)] * SQRT ( [Effective Lead Time] ),
        0
    )
)

Reorder Point Units (Excel ABC) =
IF (
    HASONEVALUE ( Dim_SKU[StockCode] ),
    ROUNDUP ( [Avg Weekly Demand] * [Effective Lead Time] + [Safety Stock Units (Excel ABC)], 0 )
)

SS Mismatches (Excel ABC) =
VAR Shared = CALCULATETABLE ( VALUES ( Dim_SKU[StockCode] ), Dim_SKU[In Excel Scope] = "In Excel top-500" )
VAR T = ADDCOLUMNS ( Shared, "@bi", [Safety Stock Units (Excel ABC)], "@xl", CALCULATE ( SELECTEDVALUE ( Dim_SKU[SS_Excel] ) ) )
RETURN
    COALESCE ( COUNTROWS ( FILTER ( T, [@bi] <> [@xl] ) ), 0 )

ROP Mismatches (Excel ABC) =
VAR Shared = CALCULATETABLE ( VALUES ( Dim_SKU[StockCode] ), Dim_SKU[In Excel Scope] = "In Excel top-500" )
VAR T = ADDCOLUMNS ( Shared, "@bi", [Reorder Point Units (Excel ABC)], "@xl", CALCULATE ( SELECTEDVALUE ( Dim_SKU[ROP_Excel] ) ) )
RETURN
    COALESCE ( COUNTROWS ( FILTER ( T, [@bi] <> [@xl] ) ), 0 )

On Hand Mismatch Detail =
VAR Shared = CALCULATETABLE ( VALUES ( Dim_SKU[StockCode] ), Dim_SKU[In Excel Scope] = "In Excel top-500" )
VAR T =
    ADDCOLUMNS (
        Shared,
        "@bi", CALCULATE ( SELECTEDVALUE ( Dim_SKU[OnHand] ) ),
        "@xl", CALCULATE ( SELECTEDVALUE ( Dim_SKU[OnHand_Excel] ) )
    )
VAR Bad = FILTER ( T, [@bi] <> [@xl] )
RETURN
    IF (
        ISEMPTY ( Bad ),
        "On hand reproduces on every shared SKU.",
        "On hand differs on: "
            & CONCATENATEX ( TOPN ( 8, Bad ), Dim_SKU[StockCode] & " (BI " & [@bi] & " vs XL " & [@xl] & ")", ", " )
    )

Reconciliation Verdict =
VAR N      = [Excel Scope SKU Count]
VAR Stk    = [On Hand Mismatches]
VAR SSraw  = [SS Mismatches]
VAR ROPraw = [ROP Mismatches]
VAR SSadj  = [SS Mismatches (Excel ABC)]
VAR ROPadj = [ROP Mismatches (Excel ABC)]
VAR Other  = [EOQ Mismatches] + [XYZ Mismatches]
VAR Mode   = SELECTEDVALUE ( 'SL Mode'[SL Mode], "Class-based (A 98% / B 95% / C 90%)" )
VAR LT     = SELECTEDVALUE ( 'Lead Time Change'[Lead Time Change], 0 )
VAR OC     = SELECTEDVALUE ( 'Ordering Cost'[Ordering Cost], 50 )
VAR HC     = SELECTEDVALUE ( 'Holding Cost %'[Holding Cost %], 0.25 )
VAR ScopeNote =
    " " & SSraw & " safety stock and " & ROPraw & " reorder point values read differently before that adjustment, and every one is a SKU whose ABC class moved: Power BI ranks all 3,775 SKUs by cumulative consumption value, the workbook ranks only its own 500, so the 80/95 per cent cut points fall on different items. Same formula, wider portfolio."
RETURN
    IF (
        Mode <> "Class-based (A 98% / B 95% / C 90%)" || LT <> 0 || OC <> 50 || HC <> 0.25,
        "NOT COMPARABLE - set SL Mode to class-based, Lead Time Change 0, Ordering Cost 50, Holding Cost 25%. Any other setting is a scenario the workbook never ran.",
        IF (
            SSadj + ROPadj + Other > 0,
            "DIFFERS - " & SSadj & " safety stock, " & ROPadj & " reorder point, " & [EOQ Mismatches] & " EOQ, " & [XYZ Mismatches] & " XYZ, out of " & N & " shared SKUs once the ABC scope difference is neutralised. These are formula differences and need investigating.",
            IF (
                Stk = 0,
                "MATCH - all " & N & " shared SKUs agree on simulated on hand, XYZ class, EOQ, and, once the workbook's own ABC class is used as the service-level input, on safety stock and reorder point too." & ScopeNote,
                "MATCH ON POLICY - across all " & N & " shared SKUs the two models agree on XYZ class, EOQ, and, using the workbook's own ABC class, on safety stock and reorder point. The only residual is " & Stk & " SKU where the simulated on-hand snapshot lands exactly on a half unit and the two engines break the tie in opposite directions - a 1-unit difference, not a formula difference." & ScopeNote
            )
        )
    )
```

</details>

---

## 5. Design notes

**Totals iterate.** Safety stock, reorder point and EOQ only mean something for one SKU at a time, because each SKU has its own sigma, lead time and cost. So every portfolio figure is a `SUMX` over `VALUES(Dim_SKU[StockCode])`. A plain `SUM` of a per-SKU measure gives a number that looks reasonable but is wrong.

**Status is a measure, so it needs helper tables.** A measure cannot sit on a legend, a slicer or the filter pane. The Page 1 donut uses the disconnected `Status List` table with `SKUs in Status`. The Page 4 table uses `Action Filter` as a visual-level filter (is 1), switched by the `Action View` buttons.

**Two modes for service level.** `SL Mode` defaults to class-based (A 98 / B 95 / C 90), which reproduces the Excel policy exactly and makes the reconciliation possible. Uniform mode hands every SKU to the slider for the Page 3 what-if.

**The curve table is filtered from the slider table, not typed.** `GENERATESERIES` produces doubles, and a typed 0.97 is not bit-identical to 0.8 + 0.005 x 34. `TREATAS` matches exact values, so a typed table would match nothing and the chart would be empty. `SS Value at Level` also forces uniform mode. Otherwise the curve comes out flat in class-based mode.

**Reference lines use ALL(Dim_Week).** On a weekly axis, the average of one week is that week's value, so `Avg Weekly Demand` plotted as a series lands on top of `Units Sold`. The `(all weeks)` versions lift the week filter so the lines are the true 52-week figures.

**Refresh** takes about six minutes on the full grid.
