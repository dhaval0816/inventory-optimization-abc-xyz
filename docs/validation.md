# Validation

This project was validated at four levels: inside Excel, between Excel and Power BI, by hand on individual SKUs, and structurally on the fact grid. The reconciliation is not a document written after the fact - it is report page 6, which reads the Excel workbook from disk on every refresh and grades itself.

**Environment:** Power BI Desktop 2.157 · `powerbi/Inventory Optimization.pbip` · run 19 Sep 2026
**Result: MATCH ON POLICY.** Across all 500 shared SKUs the two models agree on XYZ class, EOQ, and - using the workbook's own ABC class - on safety stock and reorder point. Two differences exist; both are explained below rather than patched.

---

## 1. Excel Internal Validation

**Method:** the entire policy was re-implemented from the raw `Weekly_Demand` grid, independently of the `SKU_Policy` sheet, and the two were compared column by column across all 500 SKUs.

| Column checked | Mismatches |
|---|---|
| Average weekly demand | 0 |
| Standard deviation (sample, n−1) | 0 |
| Annual demand | 0 |
| Z-score | 0 |
| Safety stock | 0 |
| Reorder point | 0 |
| EOQ | 0 |

**Summary-sheet totals reproduce to the penny:**

| Total | Value |
|---|---|
| Safety stock value | £390,415.206 |
| On-hand value | £1,208,878.458 |
| Excess value | £339,840 |
| Inventory turns | 3.19693 |
| Status counts | 108 below SS · 107 reorder now · 115 excess · 170 healthy |
| A-class below safety stock | 62 |
| Category excess table | Exact match |

**Why this check exists.** A spreadsheet that agrees with itself proves nothing if both sides came from the same formula. This re-implementation starts from the 500 × 52 raw demand grid and rebuilds every figure independently, so an error in a single cell reference or a mis-dragged formula would surface. Reproducing to three decimal places on a £1.2M portfolio means the arithmetic is right, not just consistent.

---

## 2. Excel ↔ Power BI Reconciliation

### 2.1 How the comparison is wired

This is the design decision that makes the reconciliation trustworthy:

> The workbook's `SKU_Policy` sheet is loaded as its own Power Query query (`Excel_SKU_Policy`) and left-joined to `Dim_SKU` on `StockCode`. Power BI compares itself against the real file on disk, re-read on every refresh - not against numbers copied into a document.

`Dim_SKU[In Excel Scope]` splits the portfolio into *In Excel top-500* (500 SKUs) and *Power BI only* (3,275 SKUs). Every reconciliation measure iterates only the shared 500. The joined columns (`ABC_Excel`, `XYZ_Excel`, `OnHand_Excel`, `SS_Excel`, `ROP_Excel`, `EOQ_Excel`, `SL_Excel`) are hidden from the field list and surfaced through measures so they can sit beside the live figures in one table.

**The comparison is valid only under the workbook's own assumptions**, so the verdict measure checks the parameter state first and refuses to grade otherwise:

| Parameter | Required value |
|---|---|
| SL Mode | Class-based (A 98% / B 95% / C 90%) |
| Lead Time Change | 0 weeks |
| Ordering Cost | £50 |
| Holding Cost % | 25% |

A reconciliation that silently grades under the wrong assumptions is worse than no reconciliation.

### 2.2 Results

| Check | Result | Verdict |
|---|---|---|
| Fact table shape | 3,775 SKUs × 52 weeks = 196,300 rows | Pass - no SKU missing a week |
| SKUs shared with the workbook | 500 | - |
| EOQ differs | 0 | Pass |
| XYZ class differs | 0 | Pass |
| Safety stock differs (on the workbook's ABC class) | 0 | Pass |
| Reorder point differs (on the workbook's ABC class) | 0 | Pass |
| Simulated on hand differs | 1 | Explained - §3.2 |
| ABC class differs | 210 | Expected - scope, not error - §3.1 |
| Safety stock differs, raw | 210 | Follows from ABC |
| Reorder point differs, raw | 210 | Follows from ABC |

---

## 3. The Two Differences, Explained

### 3.1 ABC class differs on 210 of 500 SKUs - a scope difference, not a bug

ABC is a relative classification: a SKU is A if it falls inside the first 80% of cumulative consumption value. Power BI ranks all 3,775 SKUs; the workbook ranks only its own top 500. Two different denominators put the 80% and 95% cut points on different items, so 210 SKUs sit in a different letter.

Safety stock inherits this directly, because the class-based service level is an ABC input:

```
Safety Stock = ROUNDUP( NORM.S.INV( ServiceLevel(ABC) ) × σ × √LT , 0 )
```

Change the letter → change Z → change the answer. Reorder point then moves with safety stock. That is why the raw mismatch count for safety stock (210) and reorder point (210) is identical to the ABC mismatch count (210) - one cause, three symptoms.

**To prove the formulas themselves agree**, the model carries a parallel set of measures that re-run identical DAX using the *workbook's* `ABC_Excel` value as the service-level input:

```dax
Class Service Level (Excel ABC) =
SWITCH ( SELECTEDVALUE ( Dim_SKU[ABC_Excel] ), "A", 0.98, "B", 0.95, "C", 0.90, 0.95 )

Safety Stock Units (Excel ABC) =
ROUNDUP (
    NORM.S.INV ( [Class Service Level (Excel ABC)] )
        * [Std Dev Weekly Demand (safe)]
        * SQRT ( [Effective Lead Time] ),
    0
)

Reorder Point Units (Excel ABC) =
ROUNDUP ( [Avg Weekly Demand] * [Effective Lead Time] + [Safety Stock Units (Excel ABC)], 0 )
```

Both return 0 mismatches. Same formula, same σ, same lead time, same rounding - the models differ only in how wide a portfolio they classify against.

**EOQ and XYZ confirm it from the other direction:** neither depends on any classification, and both reconcile outright at 0.

Classifying only the top 500 makes that scope's C class meaningless - it contains no genuinely low-value items, because they were never in the sample. The full-portfolio classification is the correct one; the workbook's is a practical limit of a formula-driven spreadsheet, and it is stated rather than smoothed over.

### 3.2 Simulated on hand differs on 1 SKU - a half-unit rounding tie

**SKU `21908` (CHOCOLATE THIS WAY METAL SIGN): Power BI 911, workbook 910.** One unit on one SKU, worth £1.26 against a £2M on-hand portfolio. Its stock status is *Healthy* either way.

On hand is simulated as `ROUND( AvgWeekly × WeeksOfCover, 0 )`. For this SKU:

```
AvgWeekly     = 3,642 ÷ 52 = 70.038461538…
WeeksOfCover  = 13  (code-derived rule)
Product       = 910.5 exactly, in decimal
```

**Nine of the 500 shared SKUs land on an exact half unit. Eight agree** - in IEEE-754 binary their product falls a hair *above* .5. For `21908` the double evaluates to `910.4999999999999`, just *below* it, and the two engines break that tie in opposite directions.

**There is nothing to fix.** The takeaway: a simulated input that sits exactly on a rounding boundary will not survive a move between tools. Any assumption column generated once and pasted, rather than recomputed identically in both engines, carries this risk.

**Why it is only one SKU and not nine:** the Power Query ribbon's *Round* uses banker's rounding (round-half-to-even), while Excel's `ROUND` rounds half away from zero. Rounding the simulated on-hand in Power Query would have disagreed with the workbook on every exact-half value. Computing it as a DAX calculated column - where `ROUND` matches Excel - leaves only this single floating-point tie.

---

## 4. Spot Checks

Three SKUs spanning the segmentation were walked through by hand with a calculator and compared across the raw grid, the workbook and the report. The workbook carries a dedicated `Spot_Check` tab for this.

| SKU | Segment (Excel 500 scope) | Checked |
|---|---|---|
| `20725` | A / X - high value, steady | Avg weekly demand, σ, Z, safety stock, ROP, EOQ |
| `22795` | B / Y - mid value, variable | Same |
| `21257` | C / Z - low value, erratic | Same |

**All three reconcile** across raw demand grid → Excel `SKU_Policy` → Power BI.

> **Class labels are scope-specific and must not be carried between tools.** `22795` is B/Y in the Excel top-500 scope and A/Y on the full 3,775-SKU Power BI scope - for exactly the reason in §3.1. The *numbers* reconcile; the *letters* legitimately differ. An earlier version of the report's page-5 note quoted the Excel letters on a full-scope page and was corrected.
>
> **General rule adopted for this project: any class or figure quoted in report text must come from the same scope the visuals use, or not be quoted at all.**

### 4.1 Worked example - page 5 default SKU

The report opens page 5 on SKU `10002` (INFLATABLE POLITICAL GLOBE), full-portfolio scope:

| Field | Value |
|---|---|
| ABC / XYZ | C / Z |
| CV | 2.28 |
| Lead time | 6 weeks |
| Safety stock | 109 units |
| Reorder point | 201 units |
| EOQ | 787 units |
| On hand | 198 units |
| Status | Reorder Now - sitting just below its reorder point |

A useful teaching case: CV 2.28 is deeply erratic, so the 109-unit buffer computed from a normal-distribution formula should be read as an order of magnitude, not a precise quantity. This is exactly the population the model recommends managing by exception rather than by rule.

---

## 5. Structural Validation

| Check | Expected | Result |
|---|---|---|
| Fact rows | 3,775 × 52 = 196,300 | 196,300 |
| Rows per SKU | exactly 52, min = max | 52 / 52 |
| Distinct weeks | 52 | 52 |
| Rows with sales | 96,033 | 49% |
| Zero-demand rows | 100,267 | 51% |
| Negative `Units` after cleaning | 0 | 0 |
| `StockCode` nulls | 0 | 0 |
| Relationship cardinality | `Dim_SKU` 1→* `Fact`, `Dim_Week` 1→* `Fact`, single direction | |
| Parameter tables | disconnected | |
| Source row count vs UCI published | 1,067,371 | |


---

## 6. Two Bugs This Exercise Found

Both were latent. The measures existed but had never been placed on a visual - so nothing had ever compiled them.

| Measure | Problem | Fix |
|---|---|---|
| `Grid Check` | `VAR Rows = COUNTROWS ( … )` - `ROWS` is a reserved word in DAX | Renamed to `RowsInGrid` |
| `Reconciliation Verdict` | `VAR Scope = …` - `SCOPE` is a reserved word in DAX | Renamed to `ScopeNote` |

**Takeaway.** Power BI accepts a model containing an uncompilable measure and says nothing until something asks for its value. A measure that is never placed on a visual is never validated. Building the reconciliation page is what forced every measure to be evaluated.


---

## 7. How to Reproduce This Validation

1. Open `powerbi/Inventory Optimization.pbip` in Power BI Desktop and Refresh (≈6 minutes on the full grid).
2. Go to page 6 - Excel Reconciliation.
3. Leave every parameter at its default, or set them to the table in §2.1.
4. Read the Verdict card. Screenshot it.
5. Use the Scope slicer to switch between the shared 500 and the full 3,775.
6. Cross-check three SKUs against the workbook's `Spot_Check` tab.

**If the workbook has moved:** Home → Transform data → Manage Parameters → `ExcelModelFile`. One value to change; nothing else in the model references a path.

---

## 8. Validation Summary

| Layer | Method | Result |
|---|---|---|
| Excel internal | Full independent re-implementation from the raw grid | 0 mismatches / 500 SKUs / 7 columns; totals to the penny |
| Excel → Power BI | Live join to the workbook on disk, re-read every refresh | 0 mismatches on EOQ, XYZ, safety stock and ROP (on the workbook's ABC) |
| Spot check | 3 SKUs by hand across all three layers | Reconciled |
| Structural | Grid shape, cardinality, nulls, zero-fill | All pass |
| Explained variances | ABC on 210 SKUs (scope) · on hand on 1 SKU (float tie) | Root-caused, documented, not patched |

