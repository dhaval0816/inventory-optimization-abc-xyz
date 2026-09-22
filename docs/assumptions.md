# Assumptions

**Every assumption in this model is stated, sourced and reproducible.** Three inputs the safety stock formula requires - unit cost, supplier lead time and on-hand inventory - do not exist in the UCI Online Retail II dataset. They are simulated under deterministic published rules, and labelled as simulated on the `Assumptions` sheet of the workbook, in the Power BI report footer, in the data dictionary and here.

> **The standard this document is written to:** simulated data in a portfolio model is acceptable and normal. *Unlabelled* simulated data is not. Anyone should be able to re-run these rules and get byte-identical values, and anyone quoting a figure from this model should be able to see immediately which inputs behind it were real.

**Real vs simulated, at a glance:**

| Input | Status | Source |
|---|---|---|
| Weekly demand quantities | Real | 500,376 cleaned transaction lines |
| Selling price | Real | Source `Price` column |
| Product description, SKU codes | Real | Source |
| Category | Derived | Keyword rule on real descriptions |
| Unit cost | Simulated | Median price × assumed margin |
| Supplier lead time | Simulated | Deterministic rule on SKU code |
| On-hand inventory | Simulated | Deterministic rule on SKU code |
| Ordering cost, holding cost | Parameter | Industry-typical, user-adjustable |
| Service level targets | Policy | Set by ABC class |

---

## 1. Demand Assumptions

### 1.1 Demand history window

| Assumption | Value |
|---|---|
| Window | 52 complete Monday-start weeks, 06 Dec 2010 → 28 Nov 2011 |
| Lines in window | 500,376 |
| Grain | SKU × week |

**Why one year, and why this year.** 52 weeks captures exactly one full seasonal cycle - critical for a giftware business whose Q4 is structurally different from the rest of the year. The dataset runs to 09 Dec 2011, but that trailing stub is a partial week; including it would understate demand for every SKU selling in it and pull σ down. The 2009-10 sheet is excluded from the modelling window so that the demand history is one continuous, complete, comparable year.

**Limitation:** one year supports a seasonal profile but not a year-on-year trend. A SKU whose demand is structurally declining is treated identically to a stable one. With two full years the model could separate trend from noise; with one, σ absorbs both.

### 1.2 Zero-demand weeks are real demand information

| Assumption | Value |
|---|---|
| Treatment | Weeks with no sale are included as `Units = 0` |
| Rows created | 100,267 of 196,300 (51%) |

σ is calculated over all 52 weekly values including zeros. Calculating it over only the 96,033 weeks with sales would measure the variability of *selling weeks* rather than the variability of *demand*, understating σ and therefore understating safety stock on precisely the intermittent items that most need a buffer. This single decision changes the answer more than any other in the model.

### 1.3 Demand is assumed normally distributed

| Assumption | Value |
|---|---|
| Distribution | Normal |
| Used by | `Safety Stock = NORM.S.INV(SL) × σ × √LT` |

**This is the model's weakest assumption, and it is stated rather than buried.** With 51% zero weeks and 89.9% of SKUs classified Z (CV > 1.0), weekly demand here is intermittent and right-skewed, not normal. The classical formula will misstate the buffer on those items - generally *over*-buffering low-volume erratic SKUs and *under*-buffering ones with rare large wholesale orders.

It is retained for three reasons, in order of importance:

1. It is the industry-standard baseline every planner recognises, and the deliverable has to be usable by the operations team that receives it.
2. The XYZ classification is what identifies where it fails. The model does not assume the formula works everywhere - it explicitly labels the 89.9% of the portfolio where it should not be trusted unattended. That is the analysis, not a gap in it.
3. The correct alternative - Croston / Syntetos-Boylan Approximation for intermittent demand - is scoped as the highest-priority enhancement and named in the README.

**How to read a Z-class safety stock number:** as an order of magnitude and a relative ranking, not a precise reorder quantity. That is exactly why the recommendation for AZ items is a weekly planner review rather than automation.

### 1.4 Demand is independent week to week

No autocorrelation, promotional lift or trend adjustment is applied. σ is the raw sample standard deviation of the 52 weekly values. In a giftware business with a strong Q4, part of what this σ measures is seasonality, not uncertainty - which inflates safety stock for the ten months of the year that are not Q4. A seasonality-adjusted safety stock is in Future Enhancements.

### 1.5 Demand = sales

The model treats units sold as units demanded. Unmet demand is invisible - an item that stocked out and lost a sale records lower demand, not higher, so the model under-buffers exactly the items that already failed. This is the classic censored-demand problem, and it cannot be solved from transaction data alone; it needs stockout event logs, which the source does not contain.

---

## 2. Lead Time Assumptions

### 2.1 Lead time is simulated 

The source has no supplier or lead time data. Lead times are assigned by a deterministic rule on the SKU code, so every re-run produces identical values:

```
Lead Time (weeks) = INDEX( {2, 4, 6, 8}, MOD( first 5 digits of StockCode, 4 ) + 1 )
```

| Lead time | Interpretation |
|---|---|
| 2 weeks | Domestic / stocked supplier |
| 4 weeks | Regional supplier |
| 6 weeks | Long-haul import |
| 8 weeks | Long-haul import with consolidation |

**Why a rule instead of a random number.** `RANDBETWEEN` would produce a different model every time the file recalculated, making the published figures unverifiable. A hash of the SKU code is reproducible, spreads the four buckets evenly across the portfolio, and carries no correlation with demand - so it does not accidentally manufacture a relationship between lead time and volume that would flatter the analysis.

**Units are weeks throughout.** Demand is weekly, lead time is weekly, √LT is in weeks. A common error in safety stock models - weekly demand multiplied by a lead time expressed in days - is designed out by never introducing a second unit anywhere in the model.

### 2.2 Lead time is deterministic - no lead-time variability

| Assumption | Value |
|---|---|
| σ of lead time | 0 (assumed perfectly reliable suppliers) |
| Formula used | `SS = Z × σ_demand × √LT` |
| Formula not used | `SS = Z × √( LT × σ_demand² + d̄² × σ_LT² )` |

**Real suppliers are not perfectly reliable, so this understates safety stock.** The extended formula is the single highest-value upgrade to the model and is named first in Future Enhancements. It is omitted from v1 because the dataset supports no basis whatever for estimating σ_LT - inventing a second layer of simulated variability on top of simulated lead times would add complexity without adding information.

**How the model compensates in the meantime:** a Lead Time Change what-if parameter (−2 to +4 weeks) lets any user stress the entire portfolio and read the capital impact directly. A +2 week shock across all suppliers adds £76,200 (+19.6%) of safety stock and £224,843 of reorder-point inventory in the Excel top-500 scope - a £301,043 working-capital call in total.

This is a live Canadian supply chain question, not an academic one. With 2026 tariff activity and cross-border delays reshaping replenishment from US and offshore suppliers, a two-week slip is an operating reality. Note the asymmetry the model exposes: safety stock scales with √LT while cycle stock scales linearly with LT, so the reorder-point exposure is roughly three times the safety-stock exposure. A business that stress-tests only its buffer will understate the working-capital impact of a lead-time extension by a factor of three.

### 2.3 Minimum effective lead time

`Effective Lead Time = MAX(1, base lead time + lead time change)`. The floor of one week prevents the −2 week scenario from driving a 2-week SKU to zero, which would return a safety stock of zero and imply that a perfectly reliable, instantaneous supplier needs no buffer.

---

## 3. Cost Assumptions

### 3.1 Unit cost is simulated 

```
Unit Cost = Median Selling Price × (1 − 40%)  =  Median Selling Price × 0.60
```

| Parameter | Value | Where set |
|---|---|---|
| Assumed gross margin | 40% | `Assumptions` sheet, yellow input cell |

**Median, not mean.** A single discounted wholesale line would drag a mean price well below the item's normal selling price; the median is robust to it.

**One stated margin instead of per-SKU costs.** A single, visible, adjustable margin assumption is auditable and reversible - change one cell and every cost, EOQ and inventory value in the model moves consistently. Invented per-SKU costs would be untraceable and would create false precision.

**What this affects:** all inventory *values* (on hand, safety stock, excess, annual consumption value), EOQ, and the ABC classification - because ABC ranks on consumption *value*. Since the same 40% is applied to every SKU, the ABC ranking is unaffected by the margin choice (it rescales every item identically); only the absolute £ figures move. All unit counts - safety stock units, reorder point, status flags - are entirely independent of this assumption.

### 3.2 Ordering cost 

| Parameter | Value |
|---|---|
| Ordering cost | £50 per purchase order |
| Nature | What-if parameter in Power BI (£20–£100); input cell in Excel |

Represents the fully loaded administrative cost of raising, tracking and receiving one PO - buyer time, receiving, invoice matching. £50 is an industry-typical mid-range figure for a distributor of this size. It appears only in EOQ, where it sits under a square root, so EOQ moves with √(ordering cost): doubling it raises EOQ by only 41%. EOQ is structurally insensitive to this assumption, which is the usual and reassuring finding.

### 3.3 Holding cost 

| Parameter | Value |
|---|---|
| Annual holding cost | 25% of unit cost |
| Nature | What-if parameter in Power BI (15%–35%); input cell in Excel |

Covers cost of capital, warehousing, insurance, shrinkage and obsolescence. 25% is the standard planning figure and sits mid-range for giftware, where obsolescence and seasonality are meaningful. Like ordering cost it appears only in EOQ, under the square root.

**Not included in the 25%:** carbon cost and the working-capital opportunity cost of the specific business. Both would raise the rate and therefore *lower* EOQ - pushing the model toward smaller, more frequent orders. A sensitivity view on the holding rate is in Future Enhancements.

### 3.4 Currency

All values are GBP (£), as sourced. No conversion to CAD is applied, because a conversion would introduce an exchange-rate assumption with a date on it and would obscure rather than clarify the source. Percentages, ratios, turns and days on hand are currency-independent and transfer directly to any market.

---

## 4. Inventory Position Assumptions

### 4.1 On-hand inventory is simulated 

The source is transactional and contains no stock positions. On hand is generated by a deterministic hash of the SKU code - never `RANDBETWEEN`, which would produce a different answer on every recalculation and make the published figures unverifiable:

```
h              = MOD( first 5 digits of SKU × 37 + LEN(SKU) × 11 , 100 )

Weeks of Cover = IF h < 20   →  1 + MOD(h, 4)      ' understocked  (~20% of SKUs)
                 IF h < 75   →  6 + MOD(h, 11)     ' healthy       (~55% of SKUs)
                 ELSE        → 26 + MOD(h, 27)     ' overstocked   (~25% of SKUs)

On Hand        = ROUND( Average Weekly Demand × Weeks of Cover , 0 )
```

The three bands deliberately produce a realistic mix of understocked, correctly stocked and overstocked items, which is what makes the status classification and the action list meaningful. A uniformly random on-hand figure would produce a flat, uninformative status distribution.

In the workbook this column is pasted as values so it stops recalculating, and the generating formula is published on the `Assumptions` sheet beside it - the value is frozen, but its provenance is not hidden.

**What this means for reading the results.** The 2,099 SKUs flagged for action, the £371K of excess and the 1,308 items below safety stock are structurally realistic but not a real stock position. The *method* is the deliverable; the *counts* demonstrate the method. Plugging in a real ERP stock file changes every count and no formula. That distinction is stated on every report page carrying a status figure.

### 4.2 Maximum stock level

```
Max Stock Level = Reorder Point + EOQ
Excess          = MAX( 0, On Hand − Max Stock Level ) × Unit Cost
```

The highest inventory position defensible under the policy: full cover through the lead time, plus the buffer, plus one economic order. Anything above it is capital the policy cannot justify. This is a conservative definition of excess - it makes no allowance for in-transit stock or committed customer orders, and a live implementation should net those off before issuing a stop-buy instruction.

### 4.3 Single stocking location

One echelon, one warehouse. No transit stock, no inter-site transfers, no multi-echelon pooling. Pooling across locations would materially reduce total safety stock (the square-root law of inventory consolidation), so this model's safety stock figure is an upper bound for a multi-site operation.

### 4.4 No minimum order quantities, batch sizes or supplier constraints

EOQ is returned as a pure economic quantity. Real suppliers impose MOQs, case packs, pallet quantities and price breaks that would round it upward. The EOQ figures here are therefore a theoretical target, not a purchase order quantity - a constraint layer sits between this model and a live ERP.

---

## 5. Classification Assumptions

### 5.1 ABC cut-offs

| Class | Cumulative consumption value share | Portfolio result (Excel top-500 scope) |
|---|---|---|
| A | ≤ 80% | 290 SKUs (80.0% of value) |
| B | ≤ 95% | 146 SKUs |
| C | > 95% | 64 SKUs |

The conventional 80/95 Pareto split. Both cut-offs are input cells, not constants, so the classification can be re-cut without touching a formula.

**ABC is ranked on annual consumption value** - `annual units × unit cost` - not on revenue, not on margin, and not on units. Consumption value is the money actually flowing through the item, which is what inventory policy exists to control.

**ABC is relative to its denominator, and this matters.** Power BI classifies all 3,775 SKUs; the Excel workbook classifies only its own top 500. The same SKU can legitimately be A in one and B in the other - 210 of the 500 shared SKUs land in a different letter. This is a scope difference, not an error, and it is analysed in full in [`validation.md`](validation.md). The practical consequence: classifying only the top 500 makes that scope's "C" class meaningless, because genuinely low-value items were never in the sample. Any ABC figure quoted from this project must carry its scope.

### 5.2 XYZ cut-offs

| Class | CV of weekly demand | Portfolio result (3,775 SKUs) |
|---|---|---|
| X | ≤ 0.5 | 7 SKUs (0.19%) |
| Y | ≤ 1.0 | 373 SKUs (9.9%) |
| Z | > 1.0 | 3,395 SKUs (89.9%) |

Conventional cut-offs, both input cells. CV is unit-free, so a £2 item and a £200 item are judged on the same scale.

**σ is the sample standard deviation (`STDEV.S`, n−1), over all 52 weeks including zeros.** The sample form is used because 52 weeks is a sample of the item's demand process, not its population.

**The result is the project's central finding.** Seven steady SKUs in 3,775. Classical reorder-point automation is safe for 0.19% of this catalogue, and any recommendation that assumes otherwise is wrong before it starts.

### 5.3 Service level by ABC class

| Class | Target cycle service level | Z-score |
|---|---|---|
| A | 98% | 2.054 |
| B | 95% | 1.645 |
| C | 90% | 1.282 |

Differentiated because the cost of a stockout is not the same on a tail item as on a revenue driver. All three are input cells, and a `SL Mode` toggle in the report switches the whole portfolio to a single uniform service level driven by a slider (0.80–0.995) when the question is "what would one more point of service cost".

**This is *cycle* service level** - the probability of not stocking out during a replenishment cycle - not fill rate. The two are often confused. Fill rate (the fraction of demand met from stock) is typically higher than cycle service level for the same buffer, because most cycles that stock out do so only near the end of the cycle and only partially.

### 5.4 Category

Assigned by a whole-word keyword match on the real product description: `BAG` → Bags, `CANDLE`/`LIGHT` → Lighting, `MUG`/`CAKE`/`TIN` → Kitchen, `CARD` → Cards/Gifts, otherwise Other.

Whole-word matching is deliberate: a naive substring test puts *PAINTING* into Tins (contains `TIN`) and *FLIGHT* into Lighting (contains `LIGHT`). The description is padded with spaces before matching so only complete words hit.

The rule is coarse - Other is the largest category and holds £225,219 of the excess - but it is transparent and it makes category-level intervention possible, which SKU-by-SKU review does not. A real implementation would use the merchandising hierarchy from the ERP.

---

## 6. Assumption Sensitivity - what actually moves the answer

| Assumption | Effect if changed | Sensitivity |
|---|---|---|
| Zero-demand weeks included | σ collapses, safety stock falls sharply on intermittent items | Extreme - changes the conclusion |
| Normal demand distribution | The whole safety stock formula changes for 89.9% of SKUs | Extreme - bounds the model's validity |
| Service level targets | 95→98 costs +£82,081 (+24.9%); 95→99 costs +£136,802 (+41.4%) | High - and non-linear |
| Lead time | +2 weeks = +£301,043 working capital (SS + ROP inventory) | High |
| Gross margin (unit cost) | All £ values scale; no unit count and no ABC rank changes | Medium - scales, does not distort |
| ABC / XYZ cut-offs | Moves SKUs between policy cells; portfolio totals barely move | Medium |
| Holding cost % | EOQ moves with 1/√(rate) | Low |
| Ordering cost | EOQ moves with √(cost) - double it, EOQ rises 41% | Low |
| On-hand simulation | Changes every status count; changes no formula | Medium - affects counts, not method |
| Category keyword rule | Redistributes excess between categories; portfolio total unchanged | Low |

**Read this table before quoting any number from this project.** The two extreme-sensitivity rows are the ones that would change a business decision, and both are methodological rather than parametric - which is why they are argued at length above rather than listed as a value.

---

## 7. What this model is not

- **Not a forecast.** It sizes buffers against historical variability; it does not predict next week's demand.
- **Not a replenishment system.** It produces a policy and an action list, not purchase orders.
- **Not tuned to a live stock position.** On hand is simulated; the counts demonstrate the method.
- **Not multi-echelon.** Single location, no pooling benefit.
- **Not constrained.** No MOQs, case packs, shelf life, supplier capacity or budget ceiling.
- **Not valid for intermittent items without caveat.** 89.9% of SKUs are Z-class; the normal-distribution formula is a baseline for them, not an answer.

Every one of these is a deliberate scope decision, and each has a named successor in the Future Enhancements section of the [README](../README.md).
