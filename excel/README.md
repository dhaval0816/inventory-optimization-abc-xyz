# `excel/` - Inventory Policy Calculator

**`inventory_policy_calculator.xlsx` - 8 sheets, 500 SKUs, every figure a live formula.**

The transparent, auditable half of this project. Power BI makes the model interactive; this workbook makes it checkable - every number can be traced back through the formula bar to a demand cell and a labelled assumption. That is why the reconciliation in [`../docs/validation.md`](../docs/validation.md) runs against this file rather than against a copy of its output.

**Scope:** top 500 SKUs by annual revenue (67.5% of portfolio revenue) × 52 weeks. This is the practical limit of a formula-driven workbook; Power BI carries all 3,775.

---

## Design conventions

One rule, applied everywhere: you can tell what a cell is by looking at it.

| Convention | Meaning |
|---|---|
| Orange text on light-yellow fill | Input. The only cells a user should type in. All of them live on `Assumptions` |
| Black text | Formula. Do not overtype |
| Dark green text | A cross-sheet link |
| Charcoal header rows, gold accent rules | Section structure |
| Purple | Reserved for the *Excess* status |

**Currency is £ (GBP)** throughout - the source dataset is a UK retailer, and converting would introduce an exchange-rate assumption with a date on it.

**Formulas, not values.** The single exception is the simulated `On Hand` column, which is pasted as values so it stops recalculating - and its generating formula is published on the `Assumptions` sheet beside the label.

---

## Sheets

### 1. `Assumptions` - the control panel

Every input in the model, in one place, each with a label and a source note.

| Input | Default | Type |
|---|---|---|
| Gross margin (drives unit cost) | 40% | Simulated |
| Ordering cost per PO | £50 | Parameter |
| Annual holding cost | 25% of unit cost | Parameter |
| ABC cut-offs | 80% / 95% | Policy |
| XYZ CV cut-offs | 0.5 / 1.0 | Policy |
| Service level - A / B / C | 98% / 95% / 90% | Policy |
| Lead-time shock | 0 weeks | Scenario |

Also on this sheet:

- The lead-time bucket table `{2, 4, 6, 8}` that `Weekly_Demand` indexes into.
- The category keyword table driving the whole-word description match.
- The published generating formula for the simulated `On Hand` column.
- A simulated-input register stating plainly that unit cost, lead time and on hand are not in the source dataset.

**Change one yellow cell and the entire workbook re-solves.** Nothing downstream is hardcoded.

### 2. `Weekly_Demand` - the data layer

500 rows × (6 attribute columns + 52 week columns), pasted from `../data/clean/weekly_demand_wide.xlsx` starting at row 5.

| Column | Source |
|---|---|
| `SKU`, `Description` | Real |
| `Category` | Formula - whole-word keyword match against the `Assumptions` table |
| `Unit Cost` | Formula - median price × (1 − margin) |
| `Lead Time (weeks)` | Formula - `INDEX(bucket table, MOD(first 5 digits, 4) + 1)` |
| `On Hand` | Pasted values - generating formula published on `Assumptions` |
| `W1` … `W52` | Real units sold, zeros included |

> **Column order is not cosmetic.** `SKU_Policy` references by position. Inserting or reordering a column breaks every downstream formula.

**Paste as values**, never as a linked range - a live link to a closed source file is the most common way this kind of workbook breaks on another machine.

### 3. `SKU_Policy` - the model

One row per SKU, one column per calculation. This sheet *is* the inventory policy.

| Column | Formula | What it answers |
|---|---|---|
| Avg weekly demand | `AVERAGE(W1:W52)` | Base consumption rate |
| Std dev weekly | `STDEV.S(W1:W52)` | Demand variability (sample, n−1) |
| CV | `σ ÷ mean` | How unpredictable, independent of volume |
| Annual demand | `SUM(W1:W52)` | Volume for EOQ |
| Annual consumption value | `annual units × unit cost` | The ABC ranking basis |
| Cumulative value share | Running total ÷ total | Where the 80/95 cut points fall |
| ABC | `≤80% → A · ≤95% → B · else C` | Where the money is |
| XYZ | `CV ≤0.5 → X · ≤1.0 → Y · else Z` | How forecastable |
| Service level | `SWITCH` on ABC | Policy input |
| Z-score | `NORM.S.INV(service level)` | Standard deviations of buffer |
| Safety stock | `ROUNDUP(Z × σ × SQRT(LT), 0)` | Buffer against variability during lead time |
| Reorder point | `ROUNDUP(avg × LT + SS, 0)` | Order when stock falls to this |
| EOQ | `ROUND(SQRT(2 × D × S ÷ (H × cost)), 0)` | Balances ordering and holding cost |
| Max stock level | `ROP + EOQ` | The ceiling |
| Status | Nested `IF` | Below SS / Reorder Now / Healthy / Excess |
| Excess value | `MAX(0, on hand − max) × cost` | Capital above the ceiling |
| Suggested order qty | `IF(on hand ≤ ROP, …)` | What to actually order |

**`ROUNDUP` on safety stock and ROP is deliberate** - you cannot hold 4.3 units, and rounding a buffer *down* under-protects every SKU in the portfolio.

**Ranking uses a `ROUND(value, 2)` helper.** Two SKUs with identical annual value would otherwise tie, and the cumulative share would count each twice and push past 100%. (The Power BI model solves the same problem with a `StockCode` tie-breaker.)

### 4. `ABC_XYZ_Matrix`

The 3 × 3 segmentation. SKU count, annual consumption value, share of portfolio value, and the recommended policy per cell. Updates automatically from `SKU_Policy`.

**Key cells (top-500 scope):** A 290 SKUs (80.0% of value) · B 146 · C 64 · X 7 · Y 188 · Z 305 · AZ 167 SKUs = 43.1% of value, £226,777 safety stock, 43 below it · AX only 7 · CZ 39 (£28,733 on hand, £3,430 excess).

### 5. `Service_Level_Scenario`

A one-way data table across service levels and a two-way table across service level × lead-time shock, showing total safety stock investment at each point.

| Move | Extra safety stock |
|---|---|
| 95% → 98% | +£82,081 (+24.9%) |
| 95% → 99% | +£136,802 (+41.4%) |
| Lead time +2 weeks | +£76,200 (+19.6%) safety stock, +£224,843 reorder-point inventory |

**Excel Data Tables, not copied numbers.** The scenario re-solves the whole model at each point; a hardcoded sensitivity table is a screenshot pretending to be a model.

### 6. `Summary` - the one-page result

| Metric | Value |
|---|---|
| SKUs | 500 |
| Annual consumption value | £3,864,699 |
| Simulated on-hand value | £1,208,878 |
| Safety stock value | £390,415 |
| Excess value | £339,840 (28.1%) |
| Inventory turns | 3.2x |
| Days on hand | 114 |
| Below safety stock | 108 (of which 62 A-class) |
| Reorder now | 107 |
| Excess | 115 |
| Healthy | 170 |
| Excess by category | Other £225,219 · Bags £66,278 (33.4%) |

### 7. `Spot_Check`

Three SKUs walked through by hand, column by column, against the raw demand grid and against Power BI: `20725` (A/X), `22795` (B/Y), `21257` (C/Z) - segment labels at this workbook's 500-SKU scope.

> **Class labels are scope-specific.** `22795` is B/Y here and A/Y on the full 3,775-SKU Power BI scope, because ABC is relative to its denominator. The *numbers* reconcile; the *letters* legitimately differ. See [`../docs/validation.md`](../docs/validation.md) §3.1.

### 8. `Data_Prep`

The Power Query steps that produced `Weekly_Demand`, documented in the workbook itself so the file is self-contained. Mirrors [`../powerquery/power_query_steps.md`](../powerquery/power_query_steps.md).

---

## How to use it

1. **Open and look at `Summary` first** - it is the whole model on one page.
2. **Change a yellow cell on `Assumptions`** and watch every sheet re-solve. Try the margin, then the A-class service level.
3. **Click into any cell on `SKU_Policy`** and read the formula bar. Nothing is hidden and nothing is pasted.
4. **Check the model yourself** on `Spot_Check` - three SKUs, by hand, with a calculator.

**To point it at your own data:** replace `Weekly_Demand` rows 5 onward, keeping the column order exactly. Everything else follows.

---

## Known limitations

| Limitation | Detail |
|---|---|
| 500-SKU scope | ABC is relative to its denominator, so this workbook's C class is not meaningful - genuinely low-value items were never in the sample. The full-portfolio classification in Power BI is the correct one |
| Three simulated inputs | Unit cost, lead time, on hand - all labelled, all documented in [`../docs/assumptions.md`](../docs/assumptions.md) |
| Normal-distribution assumption | Weak for the Z-class items that dominate the portfolio. Stated rather than hidden; XYZ is what flags where it applies |
| Deterministic lead time | No σ on lead time. The extended formula is the first item in Future Enhancements |
| File size | 500 × 52 formula cells plus data tables - recalculation is noticeable on older machines. Set calculation to Manual while editing assumptions in bulk |
