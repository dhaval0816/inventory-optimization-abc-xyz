# `data/raw/` - Source Dataset

**This folder is intentionally empty in version control.** The source file is 45.6 MB, exceeds GitHub's comfortable file size, and is publicly redistributable only with attribution - so it is downloaded rather than committed. `.gitignore` excludes `*.xlsx`, `*.xls`, `*.csv` and `*.zip` from this folder while keeping this README.

---

## Original Dataset Source

| Attribute | Detail |
|---|---|
| Name | Online Retail II |
| Repository | UCI Machine Learning Repository, dataset 502 |
| URL | https://archive.ics.uci.edu/dataset/502/online+retail+ii |
| DOI | [10.24432/C5CG6D](https://doi.org/10.24432/C5CG6D) |
| Creator / donor | Daqing Chen, School of Engineering, London South Bank University |
| Licence | Creative Commons Attribution 4.0 International (CC BY 4.0) |
| File | `online_retail_II.xlsx` - single workbook, two worksheets |
| Size | ≈ 45.6 MB |
| Instances | 1,067,371 invoice lines |
| Period | 01 December 2009 – 09 December 2011 |

### Business context of the source

Transactions from a UK-registered, non-store online retailer selling unique all-occasion giftware. A large share of customers are wholesalers, which is why order quantities are lumpy and why the demand profile in this project is intermittent rather than smooth. That characteristic is not noise in the data - it is the business, and it is what drives the project's central finding that only 7 of 3,775 SKUs are steady enough for unattended reorder-point automation.

### Schema

| Column | Type | Description |
|---|---|---|
| `Invoice` | Text | 6-digit invoice number. A leading `C` marks a cancellation - these rows are removed in cleaning. |
| `StockCode` | Text | Product code. Real products begin with 5 digits (e.g. `85123A`). Non-product codes (`POST`, `M`, `D`, `DOT`, `BANK CHARGES`, `AMAZONFEE`) are removed. |
| `Description` | Text | Product name. Contains nulls and inconsistent casing. |
| `Quantity` | Integer | Units per line. Can be negative (returns / adjustments). |
| `InvoiceDate` | DateTime | Transaction timestamp. |
| `Price` | Decimal | Unit selling price in GBP (£). Can be 0 or negative. |
| `Customer ID` | Integer | Nullable - roughly a quarter of rows have no customer. Not used in this project. |
| `Country` | Text | Customer country; predominantly United Kingdom. Not used in this project. |

### Worksheets

| Sheet | Period | Rows |
|---|---|---|
| `Year 2009-2010` | 01 Dec 2009 – 09 Dec 2010 | 525,461 |
| `Year 2010-2011` | 01 Dec 2010 – 09 Dec 2011 | 541,910 |

> **The two sheets overlap.** Transactions dated 1–9 December 2010 appear on both sheets - 22,523 duplicated rows. Appending the sheets without removing the overlap double-counts nine days of demand, which inflates average weekly demand and distorts every downstream safety stock figure. Removing this overlap is step 1b of the cleaning pipeline. See [`../../docs/data_quality.md`](../../docs/data_quality.md).

---

## Download Instructions

1. Open https://archive.ics.uci.edu/dataset/502/online+retail+ii
2. Click Download (top right). You receive `online+retail+ii.zip`.
3. Extract `online_retail_II.xlsx`.
4. Place it in this folder, unchanged:

   ```
   data/raw/online_retail_II.xlsx
   ```

5. **Never edit this file.** Every transformation is applied downstream in Power Query, so the raw file remains the immutable audit trail. If a number in the model is ever challenged, the answer has to be traceable back to an unmodified source.

**Mirror:** the same data is also published on Kaggle as *Online Retail II UCI* if the UCI archive is unreachable. Verify the row count is 1,067,371 before using any mirror - mirrors are sometimes pre-cleaned, which would silently change every figure in this project.

**Integrity check after download** - before running the Power Query steps, confirm:

| Check | Expected |
|---|---|
| Worksheets | 2 (`Year 2009-2010`, `Year 2010-2011`) |
| Total rows across both sheets | 1,067,371 |
| Columns | 8, named as in the schema above |
| `Price` column header | `Price` (some mirrors use `UnitPrice`) |

---

## Citation Format

**APA 7:**

> Chen, D. (2012). *Online Retail II* [Dataset]. UCI Machine Learning Repository. https://doi.org/10.24432/C5CG6D

**BibTeX:**

```bibtex
@misc{chen_online_retail_ii,
  author       = {Chen, Daqing},
  title        = {{Online Retail II}},
  year         = {2012},
  howpublished = {UCI Machine Learning Repository},
  doi          = {10.24432/C5CG6D},
  url          = {https://archive.ics.uci.edu/dataset/502/online+retail+ii},
  note         = {Licensed CC BY 4.0}
}
```

---

## Data Usage Note

- The source file is used read-only. No cleaning, sorting, typing or editing is performed in the raw workbook.
- All transformation happens in Power Query, documented click by click in [`../../powerquery/power_query_steps.md`](../../powerquery/power_query_steps.md), so the path from raw to model is fully reproducible by a third party.
- The cleaned output - a 3,775 SKU × 52 week demand grid - is published in [`../clean/`](../clean/). See that folder's README for the data dictionary.
- **All demand quantities in this project are real.** Unit cost, supplier lead time and on-hand inventory are not present in the source and are simulated under documented, deterministic rules. They are labelled as simulated on every sheet, every report page and in [`../../docs/assumptions.md`](../../docs/assumptions.md).
- `Customer ID` and `Country` are loaded but not used. No customer-level analysis is performed and no attempt is made to identify individuals.

---

## Disclaimer

This dataset is used strictly for educational and portfolio purposes. It is public, anonymised transactional data published by the UCI Machine Learning Repository under CC BY 4.0.

- The retailer behind the data is not a client, employer or partner of the author. The business scenario framing this project - the Operations Director's brief, the stated symptoms and the recommendations - is a realistic reconstruction written to demonstrate analytical method, not a record of an engagement with the company.
- Unit cost, lead time and inventory positions do not reflect the real company's economics. They are simulated inputs and must not be read as commercial information about any actual business.
- The figures in this repository are valid only under the assumptions documented in [`../../docs/assumptions.md`](../../docs/assumptions.md). Applying this model to a live operation requires replacing all three simulated inputs with real ERP data.
- No warranty is offered as to fitness for any commercial purpose.

Attribution is provided as required by CC BY 4.0. If the dataset licence changes, this repository will follow it.
