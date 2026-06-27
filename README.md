# Part 1 — Business Data Cleaning, Validation & Excel Reporting

## Problem Summary

A retail company exports order-level sales data from multiple internal systems into a single file. Because the data comes from different source systems, it arrives with inconsistent text formatting, mixed date formats, duplicate records, missing values, invalid discounts, sales/profit calculation mismatches, and inconsistent order/payment statuses.

This project cleans and validates that raw export, documents every issue found and every cleaning decision made, and produces a set of Excel reports a business reviewer can use directly — without needing to know anything about the cleaning process behind them.

## Dataset Description

- **File:** `data/raw_orders.xlsx`, sheet `raw_orders`
- **932 order-level records, 21 columns**, including order/ship dates, customer and location details, product category/sub-category, ship mode, quantity/price/discount/sales/cost/profit, and payment/order status.
- A second sheet, `business_rules`, ships with the source file and summarizes the business rules that govern this cleaning project (mirrored in detail below).

## Tools Used

- **Python (pandas, openpyxl)** for data profiling, cleaning logic, and Excel file generation.
- **LibreOffice (headless)** for formula recalculation/validation and for generating the preview screenshots.
- **Excel-native techniques** reflected in the cleaning logic: trimming/whitespace collapse (TRIM-equivalent), text standardization (SUBSTITUTE/Find-and-Replace-equivalent), multi-format date parsing, PivotTable-equivalent grouped summaries with sorting, and conditional-formatting-based flag highlighting.

## Cleaning Steps Performed

1. Cleaned all 10 text fields (`customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `ship_mode`, `payment_status`, `order_status`): removed extra/leading/trailing spaces and standardized casing to Title Case.
2. Parsed `order_date` and `ship_date` across 4 mixed formats found in the raw file and standardized both to `YYYY-MM-DD`.
3. Computed `shipping_delay_days` and flagged any record where the ship date was earlier than the order date.
4. Standardized `discount` (mixed numeric/percentage-text) into a clean decimal `cleaned_discount`, applying business rules for missing and out-of-range values.
5. Filled missing `region` and `ship_mode` with `"Unknown"` and flagged those fills.
6. Removed 20 exact duplicate rows; identified and flagged (but did not delete) 12 order_ids with genuinely conflicting duplicate data.
7. Added calculated columns: `cleaned_discount`, `calculated_sales`, `calculated_profit`, `profit_margin`, `shipping_delay_days`, `order_month`, `order_year`, `data_quality_flag`, plus supporting boolean QA flag columns.
8. Classified every record as `Clean`, `Warning`, or `Invalid` based on the issues found.

Full detail and rationale for every step is in **`outputs/cleaning_log.md`**.

## Business Rules Applied

| Rule Area | Action Taken |
|---|---|
| Missing region | Filled as "Unknown"; flagged |
| Missing ship_mode | Filled as "Unknown"; flagged |
| Missing discount | Treated as 0 only because quantity/unit_price were valid; flagged |
| Negative discount | Flagged as invalid |
| Discount above 30% | Flagged as invalid |
| Cancelled orders | Excluded from completed-sales summary |
| Failed payments | Excluded from completed-sales summary |
| Refunded orders | Summarized separately |
| Ship date before order date | Flagged as invalid shipping record |

"Final completed sales" = `order_status = Completed` **AND** `payment_status = Paid`.

## Summary of Data Quality Issues Found

| Issue | Count (of 932 raw records, unless noted) |
|---|---|
| Missing region | 26 |
| Missing ship_mode | 22 |
| Missing discount | 18 |
| Negative discount | 16 |
| Discount above 30% | 15 |
| Ship date earlier than order date | 22 raw / 21 post-dedup |
| Exact duplicate rows | 20 |
| Duplicate order_ids (total) | 31 |
| Conflicting duplicate order_ids | 12 |
| Sales calculation mismatch | 54 (post-dedup) |
| Profit calculation mismatch | 40 (post-dedup) |
| Completed order with non-Paid payment status | 2 |

**Final record classification (912 records after dedup):** 756 Clean / 83 Warning / 73 Invalid.

Full breakdown by category is in **`outputs/data_quality_report.xlsx`**.

## Summary of Final Pivot Reports

`outputs/pivot_summary.xlsx` contains 6 sheets, each formatted as a sortable/filterable Excel table:

1. **Sales and Profit by Region** — sorted by sales descending
2. **Sales and Profit by Category and Sub-Category**
3. **Order Count by Ship Mode** — sorted by order count descending
4. **Profit Margin by Customer Segment** — sorted by margin descending
5. **Refunded/Cancelled/Failed Orders by Region**
6. **Monthly Sales Trend** — chronological

## Key Business Insights

- **Final completed sales total ₹60.10 lakh** (`calculated_sales` summed across 602 completed + paid orders), with an overall profit margin of **~29.5%**.
- **South** region generates the highest completed sales (₹16.4L) but has one of the **lowest** profit margins (28.5%); **East** and **North** are smaller in volume but more profitable per rupee of sales (~30.2%/30.1% margin).
- **Technology** is the top-performing category by completed sales (₹22.2L), ahead of Furniture (₹19.3L) and Office Supplies (₹18.6L); within Technology, **Copiers** is the single best-performing sub-category.
- **Home Office** is the most profitable customer segment by margin (30.3%), narrowly ahead of Consumer and Corporate; **Small Business** has the lowest margin (28.6%) despite having the highest total sales among segments.
- Across all 912 cleaned records, **~16% of orders are Cancelled, ~8% Refunded, and ~8% have Failed payment** — meaning roughly **1 in 3 orders never converts to recognized revenue**, which is a meaningful exposure worth investigating with the source systems.
- **"Same Day" shipping has the highest average shipping delay (~4.2 days)** of any ship mode in the cleaned data — a likely data quality or fulfillment-process problem worth flagging to operations, since "Same Day" should not average a multi-day delay.

## Assumptions and Limitations

- Valid discount range was set at **0%–30%**, inferred from a clear gap in the data between legitimate tiers (0–25%) and clearly invalid values (55%+ and all negatives) — not stated explicitly in the source instructions.
- **12 conflicting duplicate order_ids remain in the cleaned file**, flagged but not resolved, since there's no reliable way to know which version is correct without consulting the source systems. Pivot totals do not exclude them.
- Calculation mismatches (sales/profit) and invalid discounts were preserved and flagged rather than silently corrected.
- No external validation was performed against a master list of valid city/state/customer combinations.

Full assumptions and limitations are documented in **`outputs/cleaning_log.md`**.

## Screenshots Included

- `screenshots/raw_data_preview.png` — preview of the raw, unstandardized data
- `screenshots/cleaned_data_preview.png` — preview of the cleaned, analysis-ready data with calculated columns and quality flags
- `screenshots/pivot_summary_1.png` — Sales and Profit by Region (sorted)
- `screenshots/pivot_summary_2.png` — Sales and Profit by Category and Sub-Category
