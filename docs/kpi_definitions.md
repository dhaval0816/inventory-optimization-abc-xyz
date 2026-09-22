# KPI Definitions

Every KPI in this project, with its formula in both Excel and DAX, its business interpretation, the direction it should move, and the mistakes that make it lie.

**Reading conventions**

- All demand and lead times are in weeks. No other time unit appears anywhere in the model.
- All values are GBP (£) at assumed cost - see [`assumptions.md`](assumptions.md).
- **Scope is stated on every figure.** Power BI = all 3,775 SKUs; Excel = top 500 by value (67.5% of revenue). The two are never mixed in one statement.
- Nothing below is a hardcoded number. Every KPI is a live Excel formula and a live DAX measure.

---

## 1. Average Weekly Demand

**The mean number of units consumed per week over the 52-week history, including weeks with no sales.**

```
Average Weekly Demand = Total units over 52 weeks ÷ 52
```

| | |
|---|---|
| Excel | `=AVERAGE(W1:W52)` across the 52 week columns on `Weekly_Demand` |
| DAX | `Avg Weekly Demand = AVERAGEX ( VALUES ( Dim_Week[WeekStart] ), COALESCE ( [Units Sold], 0 ) )` |
| Unit | units / week |
| Grain | per SKU |
| Direction | Neither good nor bad - it is a scale input |

**Business interpretation.** The base rate of consumption. It drives the cycle-stock half of the reorder point (`Avg Weekly Demand × Lead Time`) and the annual demand figure inside EOQ. Everything downstream inherits its accuracy.

**Where this KPI lies:** if zero-demand weeks are excluded, the denominator becomes "weeks with sales" rather than 52, and the average is inflated - on a portfolio where 51% of SKU-weeks are zero, potentially by 2×. The `COALESCE(..., 0)` in the DAX and the zero-filled grid behind it exist for this reason.

---

## 2. Demand Variability - Coefficient of Variation (CV)

**How unpredictable an item's demand is, measured independently of its volume.**

```
σ_weekly = sample standard deviation of the 52 weekly quantities (n − 1)
CV       = σ_weekly ÷ Average Weekly Demand
```

| | |
|---|---|
| Excel | `=STDEV.S(W1:W52)` and `=IFERROR(σ / AverageWeekly, 0)` |
| DAX | `Std Dev Weekly Demand = STDEVX.S ( VALUES ( Dim_Week[WeekStart] ), COALESCE ( [Units Sold], 0 ) )`<br>`CV = DIVIDE ( [Std Dev Weekly Demand], [Avg Weekly Demand] )` |
| Unit | σ in units/week; CV is dimensionless |
| Direction | Lower is better - less buffer needed |
| Portfolio value | 0.38 (aggregate); per-SKU CV drives XYZ |

**Business interpretation.** CV is the single input that decides how much buffer an item needs and whether a formula can be trusted to manage it. Because it is unit-free, a £2 item and a £200 item are judged on the same scale - which is what makes the XYZ classification comparable across a whole catalogue.

| CV | Class | What it means operationally |
|---|---|---|
| ≤ 0.5 | X | Steady - safe to automate on a reorder point |
| 0.5 – 1.0 | Y | Variable but manageable - formula plus planner review |
| > 1.0 | Z | Erratic - a formula alone will be wrong |

**Portfolio result: X 7 SKUs (0.19%) · Y 373 (9.9%) · Z 3,395 (89.9%).**

**Two traps.** (1) `STDEV.P` instead of `STDEV.S` understates σ - 52 weeks is a *sample* of the demand process, so use the n−1 form. (2) A portfolio-level CV is not the average of SKU-level CVs and must never be quoted as one; the page-5 detail card carries an explicit `HASONEVALUE` guard so it returns blank rather than silently showing the portfolio figure beside single-SKU values.

---

## 3. Safety Stock

**The buffer inventory held to absorb demand variability during the replenishment lead time.**

```
Safety Stock (units) = ROUNDUP( Z × σ_weekly × √(Lead Time in weeks), 0 )
where Z = NORM.S.INV(Service Level)
```

| | |
|---|---|
| Excel | `=ROUNDUP(NORM.S.INV(ServiceLevel) * StdDevWeekly * SQRT(LeadTimeWeeks), 0)` |
| DAX | `Safety Stock Units = IF ( HASONEVALUE ( Dim_SKU[StockCode] ), ROUNDUP ( NORM.S.INV ( [Effective Service Level] ) * [Std Dev Weekly Demand (safe)] * SQRT ( [Effective Lead Time] ), 0 ) )` |
| Portfolio total (DAX) | `Total Safety Stock Value = SUMX ( VALUES ( Dim_SKU[StockCode] ), [Safety Stock Units] * CALCULATE ( SELECTEDVALUE ( Dim_SKU[UnitCost] ) ) )` |
| Unit | units (per SKU) / £ (portfolio) |
| Direction | Lower is better at a given service level - never minimise it on its own |
| Value - 3,775 SKUs, class-based | £679K |
| Value - 500 SKUs, class-based | £390,415 |

**Business interpretation.** Safety stock is the price of uncertainty. Each of its three terms is a separate business lever:

| Term | Lever | How to reduce it |
|---|---|---|
| `Z` | Service level policy | Accept a lower service target on low-value items |
| `σ` | Demand variability | Better forecasting, customer order discipline, promotional planning |
| `√LT` | Supplier lead time | Nearer suppliers, faster transport, supplier development |

**The `√LT` term is easy to get wrong.** Variability accumulates with the *square root* of time, not linearly - doubling the lead time raises safety stock by only 41%, not 100%. That is why a lead-time shock hits cycle stock (which scales linearly) roughly three times harder than it hits the buffer.

**Rounding:** `ROUNDUP` is deliberate. You cannot hold 4.3 units, and rounding a buffer *down* systematically under-protects every SKU in the portfolio.

---

## 4. Reorder Point (ROP)

**The inventory position that triggers a replenishment order.**

```
Reorder Point (units) = ROUNDUP( Average Weekly Demand × Lead Time + Safety Stock, 0 )
                        └──── cycle stock ────┘   └─ buffer ─┘
```

| | |
|---|---|
| Excel | `=ROUNDUP(AvgWeekly * LeadTimeWeeks + SafetyStock, 0)` |
| DAX | `Reorder Point Units = IF ( HASONEVALUE ( Dim_SKU[StockCode] ), ROUNDUP ( [Avg Weekly Demand] * [Effective Lead Time] + [Safety Stock Units], 0 ) )` |
| Unit | units |
| Direction | Neither - it is a trigger, not a target |

**Business interpretation.** The operational output of the entire model. When on-hand crosses this line, place an order. The two components answer two different questions: *how much will we consume while we wait?* (cycle stock) and *how wrong could that estimate be?* (safety stock).

**Unit mismatch is the classic failure.** Weekly demand multiplied by a lead time expressed in days produces a reorder point seven times too high - and it looks plausible on a dashboard. This model holds everything in weeks and never introduces a second time unit.

**Under a lead-time shock** the ROP moves far more than safety stock does, because cycle stock scales linearly with LT. A +2 week shock across the top-500 scope adds £224,843 of reorder-point inventory against £76,200 of extra safety stock - a £301,043 total working-capital call.

---

## 5. Service Level

**The target probability of not running out of stock during a replenishment cycle.**

```
Z-score = NORM.S.INV( Service Level )
```

| | |
|---|---|
| Excel | `=NORM.S.INV(ServiceLevel)` with the class target from the `Assumptions` sheet |
| DAX | `Selected Service Level = SELECTEDVALUE ( 'Service Level'[Service Level], 0.95 )`<br>`Class Service Level = SWITCH ( SELECTEDVALUE ( Dim_SKU[ABC] ), "A", 0.98, "B", 0.95, "C", 0.90, 0.95 )`<br>`Effective Service Level = IF ( [SL Mode] = "Class-based", [Class Service Level], [Selected Service Level] )` |
| Unit | probability |
| Direction | Higher is better for the customer, lower is better for working capital - it is a policy choice, not an optimisation |

| Class | Target | Z |
|---|---|---|
| A | 98% | 2.054 |
| B | 95% | 1.645 |
| C | 90% | 1.282 |

**Cycle service level, not fill rate.** This is the probability of not stocking out *during a cycle*. Fill rate - the fraction of demand met from stock - is a different measure and is usually higher for the same buffer, because most cycles that stock out do so only near the end and only partially. Confusing the two is the most common inventory-terminology error in interviews.

**The cost of service (Excel top-500 scope, uniform level):**

| Move | Extra safety stock | Increase |
|---|---|---|
| 95% → 98% | +£82,081 | +24.9% |
| 95% → 99% | +£136,802 | +41.4% |

**The relationship is non-linear because `NORM.S.INV` is.** Z rises from 1.64 at 95% to 2.33 at 99% and then accelerates sharply - the last fraction of a percent of service costs more than the first ninety. The differentiated A/B/C policy exists precisely so that this accelerating cost is paid only where a stockout is expensive. At full portfolio scope the class-based policy implies £679K of safety stock against £583,623 for a flat 95% - roughly £95K buying protection concentrated on the A-items.

---

## 6. Inventory Coverage - Days on Hand

**How many days of consumption the current stock represents.**

```
Inventory Turns = Annual Consumption Value ÷ On-Hand Value
Days on Hand    = 365 ÷ Inventory Turns
```

| | |
|---|---|
| Excel | `=AnnualConsumptionValue / OnHandValue` and `=365 / Turns` |
| DAX | `Inventory Turns = DIVIDE ( [Annual Consumption Value], [On Hand Value] )`<br>`Days on Hand = DIVIDE ( 365, [Inventory Turns] )` |
| Unit | turns/year; days |
| Direction | Turns higher is better · Days lower is better - within service constraints |
| Value - 3,775 SKUs | 3.2x · 115 days |
| Value - 500 SKUs | 3.2x · 114 days |

**Business interpretation.** The headline working-capital efficiency measure, and the one a CFO recognises without explanation. 3.2 turns means the business cycles its entire inventory investment roughly three times a year and holds close to four months of stock.

**Is 3.2x good?** For a giftware wholesaler with lead times of 2–8 weeks, it is defensible but not strong - a comparable distributor would typically target 4–6 turns. The gap is the opportunity, and this model locates it: £371K (20.3%) of the on-hand value is above any defensible maximum stock level. Releasing it moves turns without touching service.

**Numerator and denominator must both be at cost.** Annual consumption value uses `units × unit cost`; on-hand value uses the same unit cost. Mixing a revenue-based numerator with a cost-based denominator inflates turns by the gross margin - here, by a factor of about 1.67 - and is a very common reporting error.

That the two scopes agree so closely (3.2x / 115 days vs 3.2x / 114 days) is itself a validation signal: the ratio is structurally stable across a 7.5× change in portfolio size.

---

## 7. Stockout Risk

**The count and value of SKUs currently at or below their own safety stock.**

```
Stock Status =
    On Hand ≤ Safety Stock          → "Below Safety Stock"   ← stockout risk is live
    On Hand ≤ Reorder Point         → "Reorder Now"
    On Hand >  Reorder Point + EOQ  → "Excess"
    otherwise                       → "Healthy"
```

| | |
|---|---|
| Excel | Nested `IF` on `SKU_Policy`, driven by `On Hand`, `Safety Stock` and `Reorder Point` |
| DAX | `Stock Status = SWITCH ( TRUE (), NOT HASONEVALUE ( Dim_SKU[StockCode] ), BLANK (), OnHand <= [Safety Stock Units], "Below Safety Stock", OnHand <= [Reorder Point Units], "Reorder Now", OnHand > [Reorder Point Units] + [EOQ Units], "Excess", "Healthy" )` |
| Direction | Lower is better |

**Portfolio result - 3,775 SKUs:**

| Status | SKUs | Share | Meaning |
|---|---|---|---|
| Healthy | 1,454 | 38.52% | Within policy |
| Below Safety Stock | 1,308 | 34.65% | Buffer already consumed - expedite |
| Reorder Now | 791 | 20.95% | Order today |
| Excess | 222 | 5.88% | Stop buying |
| Needing action | 2,099 | 55.6% | |

**Business interpretation.** "Below safety stock" does not mean stocked out - it means the protection against being stocked out has already been spent. It is a leading indicator, which is what makes it useful: it fires before the lost sale, not after.

**Weight it by value, not by count.** The count says 1,308 items; the risk says something sharper. In the Excel top-500 scope, 62 A-class SKUs are below safety stock, carrying £660,123 of annual consumption value - that is the number to take to a management meeting, because it converts a count into revenue at risk.

**`Stock Status` is a measure, not a column**, so it cannot be placed on a legend, a slicer or a filter. The model carries a disconnected `Status List` table plus a `SKUs in Status` measure for the status donut, and an `Action Filter` measure for the page-4 action table. This is a standard and frequently misunderstood limitation of measure-based classification.

---

## Supporting KPIs

### Economic Order Quantity (EOQ)

```
EOQ (units) = ROUND( √( 2 × Annual Demand × Ordering Cost ÷ (Holding Cost % × Unit Cost) ), 0 )
```

`EOQ Units` = `ROUND ( SQRT ( DIVIDE ( 2 * AnnualUnits * OrderCost, HoldingPct * Cost ) ), 0 )`, where `OrderCost` and `HoldingPct` are read from the two what-if sliders (defaults £50 and 25%).

The order quantity that minimises total ordering plus holding cost. Both cost inputs sit under a square root, so EOQ is structurally insensitive to them - doubling the ordering cost raises EOQ by 41%. That insensitivity is a feature: EOQ remains useful even when the cost assumptions are approximate, which they always are. EOQ assumes constant demand and no constraints, so treat it as a theoretical target, not a purchase order quantity - MOQs, case packs and pallet quantities sit between it and a real PO.

### Excess Value / Excess %

```
Max Stock Level = Reorder Point + EOQ
Excess Value    = MAX(0, On Hand − Max Stock Level) × Unit Cost
Excess %        = Excess Value ÷ On Hand Value
```

**Direction Down.** 3,775 SKUs: £371K, 20.3% of on hand. 500 SKUs: £339,840, 28.1%. The tail is proportionally *less* overstocked than the head - the overbuying happened on items people were watching. By category, Other holds £225,219 and Bags is worst proportionally at 33.4% (£66,278).

Conservative by construction: no allowance for in-transit stock or committed customer orders, so net both off before issuing a stop-buy instruction.

### Annual Consumption Value (ACV)

```
ACV = Σ(52 weeks of units) × Unit Cost
```

`Annual Consumption Value = SUMX ( Dim_SKU, CALCULATE ( [Units Sold] ) * Dim_SKU[UnitCost] )`

The ranking basis for ABC - the money that flows through an item in a year, which is what inventory policy exists to control. Excel top-500 scope: £3,864,699.

### Suggested Order Quantity

```
Suggested Order Qty = IF( On Hand ≤ Reorder Point, ROUNDUP( Reorder Point + EOQ − On Hand, 0 ) )
```

Turns the action list from a diagnosis into an instruction: order enough to bring the item back up to its maximum stock level (reorder point + EOQ). Blank where no order is needed.

---

## Aggregation Rule - the one that breaks models

**Per-SKU policy measures do not sum.** A measure evaluated over 3,775 SKUs at once has no single σ, no single lead time and no single unit cost, so `SUM([Safety Stock Units])` returns a number that is not wrong in any visible way - it is simply meaningless.

Every portfolio total in this model therefore iterates over SKUs:

```dax
Total Safety Stock Value =
SUMX (
    VALUES ( Dim_SKU[StockCode] ),
    [Safety Stock Units] * CALCULATE ( SELECTEDVALUE ( Dim_SKU[UnitCost] ) )
)
```

`SUMX` forces the measure to be evaluated once per SKU - in the row context of a single item, where σ, lead time and cost all exist - and only then adds the results. The `CALCULATE` around `SELECTEDVALUE` performs the context transition that makes the unit cost resolvable inside the iterator.

This is a common cause of a Power BI inventory model that looks right and is wrong, because the incorrect total is plausible rather than absurd. Every total in this project uses the iterator form.

---

## KPI Summary Card

| KPI | 3,775 SKUs | 500 SKUs (Excel) | Direction |
|---|---|---|---|
| On-hand value | £2.0M | £1,208,878 | |
| Annual consumption value | - | £3,864,699 | - |
| Safety stock value | £679K | £390,415 | at a given SL |
| Excess value | £371K | £339,840 | |
| Excess % of on hand | 20.3% | 28.1% | |
| Inventory turns | 3.2x | 3.2x | |
| Days on hand | 115 | 114 | |
| Below safety stock | 1,308 (34.65%) | 108 (62 A-class) | |
| Reorder now | 791 (20.95%) | 107 | - |
| Excess SKUs | 222 (5.88%) | 115 | |
| Healthy | 1,454 (38.52%) | 170 | |
| SKUs needing action | 2,099 (55.6%) | 215 | |
| X-class (automatable) | 7 (0.19%) | 7 | |

*Power BI figures are read from the report's KPI cards at the default parameter state: SL Mode = Class-based (A 98 / B 95 / C 90), Lead Time Change 0, Ordering Cost £50, Holding Cost 25%. Excel figures are the `Summary` sheet at the same assumptions.*
