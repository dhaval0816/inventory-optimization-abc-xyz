# Insights & Recommendations

**Audience:** Operations Director, Head of Procurement, Finance Business Partner
**Model state:** default assumptions - SL Mode Class-based (A 98% / B 95% / C 90%), Lead Time Change 0, Ordering Cost £50, Holding Cost 25%
**Scope:** every figure is labelled [3,775] for the full Power BI portfolio or [500] for the Excel top-500 scope (67.5% of revenue). The two are never mixed in one statement.

---

## The one-paragraph version

The business is not holding too much stock or too little. It is holding unmanaged stock - £371K sitting above any defensible maximum on 222 SKUs, while 1,308 SKUs have already consumed the buffer that was supposed to protect them. Both conditions exist simultaneously because there has never been a stocking rule to be right or wrong about. The model supplies that rule. But it also found something that changes what the rule should be: only 7 SKUs in 3,775 have demand steady enough for a classical reorder point to run unattended. This is an intermittent-demand portfolio, and the deliverable is not automation - it is a segmented allocation of planner attention, with automation reserved for the seven items that can carry it.

---

## Insight 1 - Only 7 SKUs out of 3,775 can be automated **[3,775]**

| XYZ class | SKUs | Share | Meaning |
|---|---|---|---|
| X (CV ≤ 0.5) | 7 | 0.19% | Steady - safe to automate |
| Y (0.5–1.0) | 373 | 9.9% | Variable - formula + review |
| Z (CV > 1.0) | 3,395 | 89.9% | Erratic - formula alone is unsafe |

**51% of all SKU-weeks contain no sale at all** (100,267 of 196,300).

The Excel top-500 scope returns the same absolute count of 7 - every steady item in this business is already a high-value item, and there are seven of them. Across the whole portfolio those 7 carry £91,994 of annual consumption value.

**Why this is the most important finding in the project.** Almost every inventory improvement proposal starts from "let the system reorder automatically". On this catalogue that would apply a normal-distribution formula to 3,395 items whose demand is intermittent and right-skewed - producing buffers that are confidently wrong. The XYZ classification is what makes that visible *before* the rollout rather than after it.

**What it does not mean.** It does not mean the model is unusable on Z items. It means the safety stock figure for a Z item is a relative ranking and an order of magnitude, not a precise reorder quantity - useful for prioritising attention, not for running unsupervised.

---

## Insight 2 - A third of the portfolio has already spent its buffer **[3,775]**

| Status | SKUs | Share |
|---|---|---|
| Healthy | 1,454 | 38.52% |
| Below safety stock | 1,308 | 34.65% |
| Reorder now | 791 | 20.95% |
| Excess | 222 | 5.88% |
| Needing action today | 2,099 | 55.6% |

"Below safety stock" does not mean stocked out. It means the protection against being stocked out has already been consumed - a leading indicator that fires before the lost sale.

**Weight it by value and it sharpens.** In the top-500 scope, 62 A-class SKUs are below safety stock, carrying £660,123 of annual consumption value [500]. That is the revenue-at-risk figure, and it is the number to open a management meeting with - a count of 1,308 items invites a debate about thresholds; £660K of at-risk revenue invites a decision.

---

## Insight 3 - £371K of capital is stranded above any defensible maximum **[3,775]**

| Measure | 3,775 SKUs | 500 SKUs |
|---|---|---|
| Excess value | £371K | £339,840 |
| Excess % of on hand | 20.3% | 28.1% |
| Excess SKUs | 222 (5.88%) | 115 |

Excess is defined conservatively as on hand above `Reorder Point + EOQ` - full lead-time cover, plus the buffer, plus one economic order. Anything above that line is capital the policy cannot justify.

**The comparison between scopes is the interesting part.** Excess runs at 28.1% in the top 500 but only 20.3% across the whole portfolio - the long tail is *proportionally less* overstocked than the head. That is counter-intuitive and it points somewhere specific: the overbuying happened on the items people were paying attention to. Fast movers get reordered on instinct, in round quantities, by buyers confident they will sell. The tail was simply left alone. The excess problem is not a neglect problem; it is a confidence problem, and it will not be fixed by asking buyers to look more closely at their best sellers.

**By category [500]:** Other holds £225,219; Bags is the worst proportionally at 33.4% of its own on-hand value (£66,278). Category-level intervention beats SKU-by-SKU review as an opening move - one category conversation reaches more capital than 115 item reviews.

---

## Insight 4 - AZ is where the money and the risk intersect **[500]**

| Segment | SKUs | Share of portfolio value | Safety stock | Below SS |
|---|---|---|---|---|
| AZ | 167 | 43.1% | £226,777 | 43 |
| AX | 7 | - | - | - |
| CZ | 39 | small | £28,733 on hand | £3,430 excess |

At full portfolio scope the same shape holds: A/X 7 SKUs (£91,994) against A/Y 254 SKUs (£1,739,349) [3,775].

**AZ is the cell that should own the planning calendar.** Highest value, least predictable, largest buffers, and 43 of the 167 already below those buffers. It is simultaneously the segment that most needs human judgement and the one a formula handles worst.

**The management implication is a resource decision, not an inventory one.** If a planner's week is finite, it belongs to 167 SKUs carrying 43% of portfolio value - not spread evenly across 3,775 items, and not spent on the CZ tail.

---

## Insight 5 - Service is bought at an accelerating price **[500]**

| Move | Extra safety stock | Increase |
|---|---|---|
| 95% → 98% | +£82,081 | +24.9% |
| 95% → 99% | +£136,802 | +41.4% |

At full portfolio scope, a flat 95% everywhere implies £583,623 of safety stock against £679K for the class-based policy [3,775] - roughly £95K buying protection concentrated on the A-items.

**The curve is non-linear because `NORM.S.INV` is.** Z rises from 1.64 at 95% to 2.33 at 99%, then accelerates hard. The last four points of service cost more than 41% more buffer capital than the first ninety-five.

**Why differentiation is the whole argument.** A uniform 99% target across 3,775 SKUs would spend that accelerating cost on 3,395 erratic tail items where a stockout costs almost nothing. The A 98 / B 95 / C 90 policy buys the expensive protection only where a stockout is expensive. This is the single clearest demonstration in the project that inventory policy is a resource allocation decision, not an optimisation.

---

## Insight 6 - A two-week lead-time slip costs £301K **[500]**

| Component | Impact of LT +2 weeks |
|---|---|
| Safety stock | +£76,200 (+19.6%) |
| Reorder-point inventory | +£224,843 |
| Total working capital | ≈ £301,043 |

**Note the asymmetry, because most businesses miss it.** Safety stock scales with √LT; cycle stock scales linearly with LT. So the reorder-point exposure is roughly three times the safety-stock exposure. A firm that stress-tests only its buffer will understate the working-capital impact of a lead-time extension by a factor of three.

**Canadian context.** This is not a theoretical scenario for a Canadian importer in 2026. Tariff activity, customs processing, and cross-border transport variability have made two-week replenishment slips an operating reality rather than a stress test - and a slip that lands during a peak-season build is worse than the annual average implies. Two practical consequences follow directly from the √LT relationship:

- **Nearshoring is undervalued if you only price the freight.** Moving a SKU from an 8-week to a 4-week lead time cuts its safety stock by ~29% *and* halves its cycle stock - a working-capital release that rarely appears in a landed-cost comparison.
- **The exposure should be a standing quarterly number**, not a scenario someone runs after a disruption. The Page 3 slider makes it a two-minute exercise.

---

## Insight 7 - The two models agree, and the one place they differ is instructive

Across all 500 shared SKUs, Power BI and Excel return 0 mismatches on EOQ, XYZ class, and - using the workbook's own ABC class - safety stock and reorder point.

**ABC differs on 210 of 500 SKUs**, and that difference is the insight: ABC is relative to its denominator. Ranking 500 items puts the 80% cut point on different SKUs than ranking 3,775. Classifying only a top-N subset makes that subset's C class meaningless - it contains no genuinely low-value items, because they were never in the sample. Any ABC figure quoted without its scope is not a figure.

Full detail, including the single-SKU floating-point tie on `21908`: [`validation.md`](validation.md).

---

# Recommendations

Each carries an action, an owner, an expected impact and a timeframe. Sequenced deliberately: protect revenue first, release capital second, change the operating model third.

---

### R1 - Expedite the 62 A-class SKUs below safety stock, before anything else

| | |
|---|---|
| Action | Page 4, filter ABC = A and status = Below Safety Stock. Raise purchase orders this week. Use `Suggested Order Qty` as the opening quantity, adjusted for MOQ and case pack. |
| Owner | Procurement Analyst |
| Impact | Protects £660,123 of annual consumption value [500] |
| Timeframe | This week |
| Measure of success | A-class SKUs below safety stock → 0 within one lead-time cycle |

Highest-value, lowest-effort, most reversible action in the list. 62 purchase orders against £660K of at-risk revenue.

---

### R2 - Adopt the class-based service policy (A 98 / B 95 / C 90) as company standard

| | |
|---|---|
| Action | Load per-SKU safety stock and reorder point from `SKU_Policy` into the ERP item master. Review quarterly; re-run the classification twice a year. |
| Owner | Inventory Manager |
| Impact | Replaces opinion-based ordering with a documented policy. £679K of safety stock becomes a budgeted, defended figure rather than an accident [3,775] |
| Timeframe | 4–6 weeks (ERP load and validation) |
| Measure of success | 100% of A and B SKUs carrying a system reorder point; stockout incidents on A items tracked monthly |

**The argument to Finance:** the £679K is not new spending. It is the buffer the business is *already* carrying, recalculated so that it sits on the right items. The change is its distribution, not its size.

---

### R3 - Freeze replenishment on the 222 excess SKUs; start the markdown review with Bags

| | |
|---|---|
| Action | Stop-buy list from the Page 4 Excess toggle. Category review opening with Bags (33.4% excess, £66,278), then Other (£225,219). Net off in-transit and committed customer orders before issuing any stop-buy. |
| Owner | Category Buyer, with Finance |
| Impact | Up to £371K of releasable capital [3,775] |
| Timeframe | 6–8 weeks |
| Measure of success | Excess % of on hand from 20.3% → below 12% within two quarters |

**Sequence matters.** Freezing purchase orders costs nothing and releases capital immediately. Markdown destroys margin and should follow only after a sell-through attempt at full price. And per Insight 3, the root cause is buying confidence on fast movers, not neglect of slow ones - so the corrective control belongs at the point of reorder on A-items, not in a slow-mover review.

---

### R4 - Put the 167 AZ SKUs on a named weekly planner cycle. Do not automate them.

| | |
|---|---|
| Action | A weekly review list of the AZ segment with an owner by name. Permit manual forecast override. Exclude this segment from any auto-replenishment rollout. |
| Owner | Demand Planner |
| Impact | Protects 43.1% of portfolio value [500]; targets the 43 AZ SKUs already below buffer |
| Timeframe | Immediate - it is a calendar change, not a system change |
| Measure of success | AZ SKUs below safety stock trending down month over month; zero AZ stockouts on the top 20 by value |

---

### R5 - Move the CZ tail to make-to-order or delist

| | |
|---|---|
| Action | Review the 39 CZ SKUs [500] against customer commitments and range obligations. Move survivors to make-to-order or a two-bin minimum; delist the rest. |
| Owner | Category Buyer |
| Impact | £28,733 of on-hand stock and £3,430 of excess released; more importantly, planning effort removed from items that will never repay the attention |
| Timeframe | Next range review |
| Measure of success | CZ SKU count reduced; no customer complaint traced to a delisted line |

**State the counter-argument honestly.** A CZ item may be a range-completer that a wholesale customer expects to see on the list, and delisting it can cost an order far larger than the item's own value. Check customer attachment before delisting - the financial case is clear, the commercial case is not, and that distinction should reach the decision-maker rather than being smoothed away by the analysis.

---

### R6 - Make the two-week lead-time exposure a standing quarterly figure

| | |
|---|---|
| Action | Run Page 3 at Lead Time +2 each quarter. Report the delta to Finance alongside the inventory budget. |
| Owner | Supply Chain Manager |
| Impact | Pre-quantifies a ≈£301K working-capital call [500] before it arrives unannounced |
| Timeframe | Quarterly, starting now |
| Measure of success | The figure appears in the quarterly working-capital pack |

**Why it is worth standing up as a routine.** The cost of a lead-time extension is not primarily the buffer - it is the cycle stock, which scales linearly and is three times larger. Making that visible quarterly reframes supplier proximity as a working-capital decision rather than a freight-cost decision, which is the framing that survives a conversation with Finance.

---

## What was deliberately *not* recommended

Stating what was rejected, and why, is part of the analysis:

| Not recommended | Why |
|---|---|
| Auto-replenishment across the catalogue | Only 7 of 3,775 SKUs are steady enough. Rolling it out portfolio-wide would apply a normal-distribution formula to 3,395 intermittent items and produce confidently wrong buffers |
| A uniform 99% service level | Costs +41.4% more safety stock [500] and spends it mostly on tail items where a stockout is nearly free |
| Cutting safety stock to raise turns | Turns would improve and service would collapse. 1,308 SKUs are *already* below buffer - the problem is distribution, not level |
| Capping demand outliers | The large wholesale orders are the demand pattern, not errors. Capping them suppresses σ and designs the buffer to fail on the orders it exists to cover |
| A blanket inventory reduction target | Would be met by cutting the easiest items - typically A-class fast movers where stock is genuinely needed. Segment-specific targets, or none |

---

## Reading these numbers responsibly

**Unit cost, supplier lead time and on-hand inventory are simulated** under documented, deterministic rules ([`assumptions.md`](assumptions.md)). All demand figures are real, drawn from 500,376 cleaned transaction lines.

What this means in practice:

- **The £ values are structurally realistic, not a real stock position.** Replace the on-hand column with an ERP extract and every count changes; no formula changes.
- **The ratios, percentages and segment shares are robust**, because they are driven by real demand and by a margin assumption applied uniformly to every SKU.
- **The method is the deliverable.** The counts demonstrate that the method produces actionable output - 2,099 named SKUs with a status and a suggested quantity, not a chart.

Every recommendation above is expressed as an action on a named, filterable population in the report, so each one can be re-run against real data without reworking the analysis.
