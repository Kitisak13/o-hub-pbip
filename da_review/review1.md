Ran project file listing
Ran data Agent Kit skills listing
Ran semantic model files read
Analyzed output.txt:L150-290
Analyzed expressions.tmdl:L1-291
Analyzed output.txt:L290-325
Analyzed Dim_Date.tmdl:L1-400
Analyzed Dim_Product.tmdl:L1-180
Analyzed \_Measures.tmdl:L1-800
Analyzed \_Measures.tmdl:L800-1473
Ran report pages and visuals summary
Ran report details inspection
Ran page filters and overlap check
Ran field usage analysis
Ran visual filter check
Created project_review.md
Edited project_review.md

I reviewed the semantic model, the Power Query code, the measures and the three report pages, and wrote everything up in [project_review.md](file:///C:/Users/kitisak.jun/.gemini/antigravity/brain/09d4d24d-c112-4817-911a-79af2e186b95/project_review.md). It includes fix code for each item. I only read the files: I didn't refresh the model or run anything against the data, and I haven't changed any project files.

**Fix first — these four show wrong numbers or can break refresh:**

1. **Rice groups are wrong** ([Dim_Product.tmdl](file:///d:/OneDrive%20-%20CPCRT/Job/Power%20BI%20Project/Project-Oryza-Hub/pbip/bi-rice-price.SemanticModel/definition/tables/Dim_Product.tmdl#L70-L85)). The `Benchmark Group` column checks product names in the wrong order:
   - Every "25% broken" product and "U.S. 15% broken" contain the text "5% broken", so they land in **White Rice 5%**.
   - "U.S. Calrose" is also put in White Rice 5%.
   - "ปลายข้าวหอมมะลิ" (jasmine broken rice) is put in **Hom Mali** instead of Broken Rice.
2. **The FOB-by-country chart averages every grade** ([\_Measures.tmdl](file:///d:/OneDrive%20-%20CPCRT/Job/Power%20BI%20Project/Project-Oryza-Hub/pbip/bi-rice-price.SemanticModel/definition/tables/_Measures.tmdl#L645-L682)). When no rice group is selected, the Thailand line mixes 5%, 25%, Hommali, parboiled and A1. That's why it doesn't match the "5% Latest" cards next to it.
3. **Old-crop CPI prices are counted.** `Price (THB/kg)` doesn't filter to `Price Selection = "Keep"`. Rows marked "Exclude" therefore flow into Trend Signal, the 52-week figures and the moving averages shown in both matrices.
4. **Refresh can fail.** `Dim_Product` removes duplicates across 7 columns, including the product name. If a source renames a product, the same Product Code appears twice and the relationship to the fact table fails at refresh.

**Next:**

- **Speed:** every price measure looks up the exchange rate row by row. Doing the USD/THB conversion in Power Query, once per refresh, should make the matrices much faster.
- **Hidden slicer:** on the CPI/DIT page, the Benchmark Group slicer sits underneath the line chart and the left navigation panel, so users can't see or click it.
- **Two things to check in Desktop:**
  - The CPI date conversion uses the `en-US` culture. If the sheet sends dates as Thai-format text, any day of 12 or less gets swapped with the month.
  - When a crop code can't be read, the date falls back to the current month, so that crop's New/Old status changes with the refresh date.

**Clean-up:** 36 of the 96 measures aren't used anywhere, several are duplicates of each other, and source codes are hard-coded in about 30 measures. No visual has alt text. The active theme is still called "Warehouse Utilization" (copied from another project), and there's an unused theme file.

Items 1–4 are small edits, about 30 minutes in total. Should I apply them to the TMDL files now?
