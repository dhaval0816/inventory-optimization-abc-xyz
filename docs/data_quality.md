# Data Quality Report

**Source:** UCI Online Retail II - 1,067,371 invoice lines, 01 Dec 2009 – 09 Dec 2011
**Assessment date:** September 2026
**Tooling:** Power Query (Excel + Power BI Desktop), Column Quality / Column Distribution / Column Profile with full-dataset profiling enabled
**Verdict:** Fit for purpose after cleaning. 95.1% of source lines survive quality filtering; the 4.9% removed are cancellations, adjustments and non-product service codes, each removed for a stated reason with a counted impact.

---

## 1. Cleaning Audit Trail

Every filter, in order, with the row count it produced. These counts are reproducible by following [`../powerquery/power_query_steps.md`](../powerquery/power_query_steps.md), and they are the evidence base for every figure in this project.

| # | Step | Rows out | Removed | % of source | Reason |
|---|---|---|---|---|---|
| 0 | Sheet `Year 2009-2010` | 525,461 | - | - | |
| 0 | Sheet `Year 2010-2011` | 541,910 | - | - | |
| 1 | Append both sheets | 1,067,371 | - | - | Baseline |
| 1b | Remove 1–9 Dec 2010 sheet overlap | 1,044,848 | 22,523 | 2.11% | Duplicated across both sheets |
| 2a | Remove cancellations (`Invoice` starts `C`) | 1,025,683 | 19,165 | 1.80% | Reversals, not demand |
| 2b | Remove `Quantity` ≤ 0 | 1,022,290 | 3,393 | 0.32% | Returns / stock adjustments |
| 2c | Remove `Price` ≤ 0 | 1,019,654 | 2,636 | 0.25% | Samples, giveaways, entry errors |
| 3 | Keep 5-digit product codes | 1,014,945 | 4,709 | 0.44% | Non-product service codes |
| 3b | Remove invoice `541431` | 1,014,944 | 1 | 0.00% | 74,215-unit line, fully reversed |
| - | Total removed | | 52,427 | 4.91% | |
| 5 | Keep 52 complete weeks | 500,376 | 514,568 | - | Scope decision, not a quality filter |
| 6 | Group to SKU × week | 96,033 | - | - | Aggregation |
| 7 | Zero-fill to full grid | 196,300 | +100,267 | - | Rows *added*, not removed |
| 8 | Excel top-500 scope | 26,000 | - | - | Scope decision |

**Retention rate: 95.09%.** A cleaning pipeline that discards a large share of its source usually indicates a filter that is too aggressive; one that discards almost nothing usually indicates filters that are not working. Under 5% removal, with every removal individually justified and counted, is the expected profile for transactional retail data.

---

## 2. Completeness Checks

### 2.1 Column-level completeness - source

Measured with View → Data Preview → Column quality, profiler set to entire data set (the default 1,000-row sample is meaningless on a million rows).

| Column | Empty | Error | Treatment |
|---|---|---|---|
| `Invoice` | 0% | 0% | - |
| `StockCode` | 0% | 0% | - |
| `Description` | ~0.2% | 0% | Row kept. Description drives no calculation; discarding the row would delete real demand to fix a cosmetic gap |
| `Quantity` | 0% | 0% | - |
| `InvoiceDate` | 0% | 0% | - |
| `Price` | 0% | 0% | - |
| `Customer ID` | ~22% | 0% | Column removed. No customer-level analysis is performed, so the gap is irrelevant to this project |
| `Country` | 0% | 0% | Column removed - not used |

**A missing `Description` is a labelling problem; a missing `Quantity` would be a data problem.** They are treated differently on purpose. Dropping rows for a null in a descriptive field is a common and costly habit - it silently deletes demand.

### 2.2 Grid completeness - the critical check

The single check this model depends on:

| Check | Expected | Result |
|---|---|---|
| Distinct SKUs | 3,775 | 3,775 |
| Distinct weeks | 52 | 52 |
| Fact rows | 3,775 × 52 = 196,300 | 196,300 |
| Rows per SKU (min) | 52 | 52 |
| Rows per SKU (max) | 52 | 52 |
| Rows with `Units` > 0 | - | 96,033 (49%) |
| Rows with `Units` = 0 | - | 100,267 (51%) |

**51% of all SKU-weeks contain no sale.** Those 100,267 rows do not exist in the source - they are constructed by the pivot → replace-null → unpivot round trip, and they are not padding. They are the demand signal.

**What happens if the zero-fill is skipped:** σ is computed over only the weeks in which the item sold, which measures the variability of selling weeks rather than of demand. The result is systematically understated σ, understated safety stock, and understated buffers concentrated on exactly the intermittent items that most need them. On a portfolio that is 89.9% Z-class, this is not a rounding issue - it is the difference between a model that works and one that fails quietly.

### 2.3 Temporal completeness

| Check | Result |
|---|---|
| Week sequence 06 Dec 2010 → 28 Nov 2011 | 52 consecutive Mondays, no gap |
| Partial weeks in window | 0 - both boundaries land on a Monday |
| Trailing partial week (Dec 2011) | Excluded by design |
| Weeks with no transactions anywhere | 1 - week beginning 27 Dec 2010 (Christmas closure) |

**The 27 Dec 2010 week is the trap in this dataset.** It contains no rows at all, so the pivot that builds the grid would produce no column for it and the window would quietly shrink to 51 weeks. A single zero-demand seed row is appended before the pivot to hold the week open; it carries no units and no value, and it is documented in [`assumptions.md`](assumptions.md). Without it, every standard deviation in the model would be taken over 51 observations instead of 52.

A partial period at the end of a demand history is the most common cause of an understated σ in inventory models: the short week reads as a demand collapse. It is excluded explicitly and the exclusion is a named step in the query.

---

## 3. Accuracy Checks

### 3.1 Value-range validation

| Column | Rule | Violations | Action |
|---|---|---|---|
| `Quantity` | > 0 for demand | 3,393 | Removed - returns and adjustments |
| `Price` | > 0 | 2,636 | Removed - samples and entry errors |
| `InvoiceDate` | within 2009-12-01 … 2011-12-09 | 0 | - |
| `StockCode` | first 5 characters numeric | 4,709 | Removed - service codes |
| `Units` (post-grid) | ≥ 0 | 0 | |
| `Revenue` (post-grid) | ≥ 0 | 0 | |

### 3.2 Non-product codes removed

The source mixes products with service lines. These are financial transactions, not stock movements, and would corrupt every SKU-level figure if classified as products:

| Code | Meaning |
|---|---|
| `POST` | Postage charge |
| `M` / `m` | Manual adjustment |
| `D` | Discount |
| `DOT` | DOTCOM postage |
| `BANK CHARGES` | Bank fee |
| `AMAZONFEE` | Marketplace commission |
| `C2` | Carriage |
| `S` | Samples |

**Detection without a formula:** extract the first 5 characters, convert to whole number, and Remove Errors. Any code whose first five characters are not numeric raises a conversion error and is removed. A ribbon button performing the work of an error-handling expression - this is how the pipeline stays M-free.

### 3.3 Consistency - SKU code normalisation

| Issue | Treatment | Impact if skipped |
|---|---|---|
| Leading/trailing spaces (`" 85123A"`) | Trim | The same product splits into two SKUs |
| Mixed case (`"85123a"` / `"85123A"`) | UPPERCASE | Demand history halves; average weekly demand halves; safety stock halves |

Both applied before any grouping. Grouping on an un-normalised key is the most expensive silent error in this pipeline - it produces a completely plausible model built on a fragmented SKU master.

### 3.4 Cross-source reconciliation

| Check | Method | Result |
|---|---|---|
| Source row count | Against UCI published instance count (1,067,371) | Match |
| SKU-week totals | Three SKUs (`20725`, `22795`, `21257`) summed by hand from raw source and compared to `Fact_WeeklyDemand` | Match |
| Excel ↔ Power BI policy outputs | Full re-implementation, 500 shared SKUs | 0 mismatches on safety stock, ROP, EOQ, XYZ - see [`validation.md`](validation.md) |

---

## 4. Duplicate Handling

### 4.1 The sheet overlap - 22,523 duplicated rows

**The most consequential quality issue in this dataset, and the easiest to miss.**

| Detail | Value |
|---|---|
| Overlap period | 01 – 09 December 2010 |
| Rows duplicated | 22,523 |
| Share of source | 2.11% |
| Detection | Distinct count of `Invoice` before and after append; date-range profile per sheet |

Both worksheets contain the first nine days of December 2010. A plain append counts those nine days twice.

**Why it matters more than 2% suggests.** The duplication is not spread evenly - it is concentrated in nine consecutive days that fall inside the 52-week modelling window, in the first two weeks of that window. For any SKU selling in that period, those weeks read as roughly double their true demand. That inflates the mean, inflates σ, and inflates safety stock - and it does so in a way that looks entirely plausible on a dashboard. A duplicate that produces an obviously wrong number is harmless; one that produces a believable wrong number is not.

**Treatment:** removed as an explicit, named step (`Removed sheet overlap`) before any other filter, so a reviewer can see in the Applied Steps pane that the issue was known and handled.

### 4.2 Legitimate repeated rows

The source contains rows identical in `Invoice`, `StockCode`, `Quantity` and `Price` that are not duplicates - the same product appearing twice on one invoice, typically a picking or packing split. These are kept. They are real demand, and `Remove Duplicates` applied to the transaction table would delete genuine units.

**Rule applied throughout: de-duplicate on an identified defect, never on apparent row similarity.** The only de-duplication performed on this dataset is the sheet overlap (a known structural defect) and the `StockCode` de-duplication when building the SKU dimension (where one row per SKU is by definition the required grain).

### 4.3 Grain-level uniqueness

| Table | Key | Duplicates |
|---|---|---|
| `Fact_WeeklyDemand` | `StockCode` + `WeekStart` | 0 - guaranteed by Group By, and confirmed because Pivot Column with *Don't Aggregate* would error on any duplicate |
| `Dim_SKU` | `StockCode` | 0 - Remove Duplicates applied after the most-frequent-description group |
| `Dim_Week` | `WeekStart` | 0 - 52 distinct values |

Using Don't Aggregate on the pivot is itself a duplicate test: if the grain had been violated upstream, the step fails loudly instead of silently summing the collision away.

---

## 5. Outlier Handling

### 5.1 Invoice 541431 - one row, 74,215 units

| Detail | Value |
|---|---|
| Invoice | `541431` |
| Quantity | 74,215 units on a single line |
| Reversal | Credit note `C541433`, minutes later |
| Decision | Removed |

**Why it had to be removed specifically.** The credit note was already deleted by the cancellation filter (step 2a), which strands the original as a permanent 74,215-unit phantom sale in one week. For the affected SKU that single week would dominate the mean and σ outright - producing an enormous, entirely fictitious safety stock. A matched pair of transactions must be removed as a pair or kept as a pair; removing one side is worse than removing neither.

**The generalisable rule: after filtering cancellations, look for the orphaned originals.** This applies to any transactional dataset with reversals.

### 5.2 Systematic outlier screen

**Rule applied:** flag any SKU-week where `Units > mean + 4σ` for that SKU.

| Step | Approach |
|---|---|
| Detection | Compare weekly units to each SKU's own mean + 4σ |
| Investigation | Sample the flagged weeks and inspect the underlying invoices |
| Finding | Predominantly genuine large wholesale orders - this is a wholesale business, and lumpy bulk ordering is the demand pattern, not an error |
| Decision | Keep, uncapped |

**Why keeping them is the right call here, and why it is a judgement not a default.** Capping outliers would suppress σ, which would lower safety stock on precisely the items whose demand is genuinely lumpy - designing the buffer to fail on the orders it exists to cover. The large orders *are* the risk the safety stock is sized against.

**The counter-argument, stated fairly:** if a bulk order is known in advance (a contract, a seasonal programme), it is not uncertainty and should be planned separately rather than buffered against. A production implementation would separate contracted volume from unplanned demand and buffer only the second. The dataset carries no flag that would allow that separation, so the conservative choice - keep everything, buffer against all of it - is taken and stated.

**The one exception is §5.1**, removed not for being large but for being a reversed transaction with its counterpart already deleted. The distinction matters: outliers are removed for being *wrong*, never for being *big*.

### 5.3 Zero-demand weeks are not outliers

51% of SKU-weeks are zero. They are neither errors nor outliers - they are the defining characteristic of an intermittent-demand portfolio, and they are constructed deliberately. Treating them as missing data and removing them is the single most common error in inventory analytics, and it produces a model that looks fine and under-buffers everything.

---

## 6. Known Limitations of the Source

| Limitation | Impact | Mitigation |
|---|---|---|
| No unit cost | Cannot value inventory from source | Simulated: median price × 0.60, labelled - [`assumptions.md`](assumptions.md) |
| No supplier or lead time | Cannot calculate safety stock from source | Simulated: deterministic rule, labelled |
| No on-hand inventory | Cannot flag status from source | Simulated: deterministic hash, labelled |
| No stockout events | Lost sales are invisible; demand is censored | Stated openly - the model under-buffers items that already stocked out |
| No promotion calendar | Promotional lift is absorbed into σ as if it were uncertainty | Inflates safety stock during promotional periods |
| No returns linkage | Returns removed rather than netted against the originating sale | Immaterial at 0.32% of lines |
| One year in the modelling window | No year-on-year trend separation | σ absorbs both trend and noise; stated as a limitation |
| Single country, single business model | UK wholesale giftware | Findings are methodological; the figures are not benchmarks for other sectors |

---

## 7. Data Quality Scorecard

| Dimension | Score | Evidence |
|---|---|---|
| Completeness | Strong | 0% nulls on every modelling column; 196,300-row grid complete; every SKU has exactly 52 weeks |
| Uniqueness | Strong | Sheet overlap identified and removed; grain uniqueness enforced by Group By and proven by *Don't Aggregate* |
| Validity | Strong | Type, range and pattern rules applied to every column; 4,709 non-product codes removed by rule, not by hand |
| Accuracy | Strong | 3-SKU manual reconciliation to source; full Excel ↔ Power BI reconciliation at 0 mismatches |
| Consistency | Strong | SKU codes trimmed and uppercased before any grouping |
| Timeliness | Historical | Data ends Dec 2011; used as a methodological dataset, not a current operating picture |
| Coverage | Partial | Three required inputs absent from source and simulated under labelled rules |

**Overall: fit for purpose.** The dataset supports a full inventory policy model. The three simulated inputs are the boundary of what it can prove, that boundary is documented everywhere the inputs appear, and no figure in this repository is quoted without its scope and its assumption status attached.
