# Power BI project files

This folder holds the actual Power BI Desktop project, saved in the **PBIP** format (plain text, diffable) rather than as a binary `.pbix`.

| Item | What it is |
|---|---|
| `Inventory Optimization.pbip` | The project file Power BI Desktop opens |
| `Inventory Optimization.SemanticModel/` | The data model in TMDL: queries, tables, relationships, calculated columns and all 84 measures |
| `Inventory Optimization.Report.zip` | The report definition - 96 generated JSON files across ~90 folders, zipped so the repository stays readable |
| `build_notes/` | How the model and report were built, step by step |
| `theme_inventory_teal.json` | The report theme |

## To open it

1. Download this folder.
2. Unzip `Inventory Optimization.Report.zip` **beside** the `.pbip` file, so you have:
   ```
   Inventory Optimization.pbip
   Inventory Optimization.Report/
   Inventory Optimization.SemanticModel/
   ```
3. Open `Inventory Optimization.pbip` in Power BI Desktop (December 2023 or later).
4. Point the two parameters at your own copies of the files - `SourceFile` to `online_retail_II.xlsx`, `ExcelModelFile` to the workbook in [`../excel/`](../excel) - then refresh. The source data is not in this repository; see [`../data/raw/README.md`](../data/raw/README.md).

## Where to look first

If you only read one file, read
[`Inventory Optimization.SemanticModel/definition/expressions.tmdl`](Inventory%20Optimization.SemanticModel/definition/expressions.tmdl).

That is the whole ETL. Every query in it is the M that Power Query generated from ribbon clicks - step names like `#"Filtered Rows"`, `#"Removed Errors"`, `#"Pivoted Column"`, `#"Unpivoted Other Columns"` are what the menu commands produce. There is exactly one hand-typed formula in the file, `#"Added Custom"` in `Sales_Raw`:

```m
Date.StartOfWeek([InvoiceDate], Day.Monday)
```

It is typed because the ribbon's *Start of Week* command always uses Sunday, and this dataset's weeks run Monday to Sunday. The reasoning is in [`build_notes/00_BUILD_WITHOUT_M_CODE.md`](build_notes/00_BUILD_WITHOUT_M_CODE.md).

The calculated columns and measures are in
[`Inventory Optimization.SemanticModel/definition/tables/`](Inventory%20Optimization.SemanticModel/definition/tables) - `Dim_SKU.tmdl` for the 17 classification columns, `_Measures.tmdl` for the 80 measures.
