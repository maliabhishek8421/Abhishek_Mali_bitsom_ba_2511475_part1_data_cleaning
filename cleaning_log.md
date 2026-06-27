# Cleaning Log — Retail Order Data

**Source file:** `data/raw_orders.xlsx` (932 records, 21 columns)
**Output file:** `data/cleaned_orders.xlsx` (912 records, 39 columns)
**Date:** Cleaning performed as part of Part 1 — Business Data Cleaning, Validation & Excel Reporting

---

## 1. Issues Found

### Text formatting issues
- Leading/trailing spaces and double spaces in `customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `ship_mode`, `payment_status`, `order_status` (e.g. `"  Small Business "`, `"Vikram  Iyer"`).
- Case inconsistencies across the same fields (e.g. `"NORTH"`, `"north"`, `"  North"`, `"North"` all representing one value; `"SMALL BUSINESS"` vs `"Small Business"`).
- No genuinely different spellings of the same category were found beyond casing/whitespace (e.g. no "Office Supply" vs "Office Supplies" variants) — only formatting noise, not true naming conflicts.

### Date issues
- Four different date formats were mixed within both `order_date` and `ship_date`: `DD Mon YYYY` (e.g. `21 Jul 2024`), `MM/DD/YYYY` (e.g. `08/31/2024`), `DD-MM-YYYY` (e.g. `28-11-2024`), and `YYYY-MM-DD` (e.g. `2024-05-24`).
- No missing dates and no unparseable text values were found in either date column — every value matched one of the four formats above.
- **22 records** had a `ship_date` earlier than the `order_date` (1 of those 22 was later removed as part of an exact-duplicate row, leaving **21** in the final cleaned file). These are logically invalid shipping records and were flagged, not silently corrected, since the true intended dates cannot be inferred.

### Duplicate issues
- **20 exact duplicate rows** were found (every column identical, including `order_id`) — these are pure re-exports of the same row and were removed, keeping the first occurrence.
- **31 `order_id` values** appeared more than once in the raw file.
  - Of these, **12 order_ids had conflicting data** between their duplicate rows (different `sales`, `cost`, `profit`, `order_status`, or `payment_status` values under the same order_id) — for example `ORD-2024-10124` appears three times with two different `sales`/`order_status` combinations.
  - The remaining duplicate order_ids were exact duplicates and were resolved by the exact-duplicate removal step above.

### Discount issues
- `discount` was stored in mixed formats: decimal numbers (`0.2`), whole-number zero (`0`), and percentage text (`"70%"`, `"85%"`).
- **18 records** had a missing discount value.
- **16 records** had a negative discount (e.g. `-0.19`, `-0.24`) — not business-valid.
- **15 records** had a discount above the maximum valid business range — these were ≥55% (`0.55`, `0.65`, `0.70`, `0.85`), far outside the observed legitimate tier structure of 0%, 5%, 10%, 15%, 20%, 25%.
- No values fell between 30% and 55%, confirming a clear, defensible cut-off for "valid" vs "invalid" discount.

### Calculation mismatches
- **54 records** (post-dedup) had a `sales` value that did not match `quantity × unit_price × (1 − discount)` within a tolerance of 1.0.
- **40 records** had a `profit` value that did not match `sales − cost` within a tolerance of 1.0.
- These were not overwritten — both the original raw figures and the recalculated `calculated_sales` / `calculated_profit` are kept side by side in the cleaned file so the evaluator/business user can see both and decide.

### Order/payment status issues
- `order_status` and `payment_status` both had the same casing/whitespace noise described above.
- **2 records** had `order_status = Completed` but `payment_status = Failed` — a data inconsistency, since a completed order should not have a failed payment. These were excluded from the completed-sales summary on the `payment_status` rule rather than the `order_status` rule, since a non-Paid status overrides "Completed" for revenue-recognition purposes.

---

## 2. Cleaning Actions Performed

1. **Text fields** — collapsed multiple/leading/trailing spaces, removed non-breaking spaces, and standardized casing to Title Case for all 10 categorical/text columns listed above.
2. **Dates** — parsed `order_date` and `ship_date` by trying each of the four observed formats (`DD Mon YYYY`, `MM/DD/YYYY`, `DD-MM-YYYY`, `YYYY-MM-DD`) in order, then standardized both to `YYYY-MM-DD`. Computed `shipping_delay_days = ship_date − order_date`. Flagged (did not delete) any record where this is negative.
3. **Discount** — converted percentage-text values to decimals (`"70%"` → `0.70`). Created `cleaned_discount`:
   - Missing discount → set to `0` **only if** `quantity` and `unit_price` were both present and non-negative (this was true for all 18 missing-discount records).
   - Negative or >30% discount → left blank in `cleaned_discount` and flagged `discount_invalid = TRUE` (not assumed or corrected).
4. **Missing region / ship_mode** — filled both with `"Unknown"` per the business rule, and flagged `region_missing` / `ship_mode_missing = TRUE` so these can still be identified and excluded from region/ship-mode specific analysis if needed.
5. **Duplicates** —
   - Removed the 20 extra copies of exact duplicate rows, keeping the first occurrence of each.
   - Did **not** delete any of the 12 conflicting duplicate order_ids. All versions remain in the cleaned file, each flagged `conflicting_duplicate = TRUE`, for manual business review (see `outputs/data_quality_report.xlsx` → "Duplicate Summary" for the full list of affected order_ids).
6. **Calculated columns added** (see `cleaned_orders.xlsx` → `column_reference` sheet for full descriptions):
   `cleaned_discount`, `calculated_sales`, `calculated_profit`, `profit_margin`, `shipping_delay_days`, `order_month`, `order_year`, `data_quality_flag`, plus supporting boolean flag columns (`region_missing`, `ship_mode_missing`, `discount_missing`, `discount_invalid`, `ship_before_order`, `duplicate_order_id`, `conflicting_duplicate`, `sales_mismatch`, `profit_mismatch`).
7. **Data quality flag** — every record was classified as:
   - **Invalid** — unparseable date, ship-before-order date, invalid discount, or conflicting duplicate.
   - **Warning** — missing region/ship_mode/discount that was filled per business rule, or a sales/profit figure that didn't match recalculation.
   - **Clean** — none of the above.

---

## 3. Business Rules Applied

| Rule Area | Action Taken |
|---|---|
| Missing region | Filled as `"Unknown"`; flagged `region_missing = TRUE` |
| Missing ship_mode | Filled as `"Unknown"`; flagged `ship_mode_missing = TRUE` |
| Missing discount | Treated as `0` only because `quantity` and `unit_price` were valid in all 18 cases; flagged `discount_missing = TRUE` |
| Negative discount | Flagged `discount_invalid = TRUE`; excluded from `cleaned_discount` (left blank) |
| Discount above allowed range (>30%) | Flagged `discount_invalid = TRUE`; excluded from `cleaned_discount` (left blank) |
| Cancelled orders | Excluded from the final completed-sales summary (`is_completed_sale = FALSE`) |
| Failed payments | Excluded from the final completed-sales summary, regardless of order_status |
| Refunded orders | Summarized separately (see `pivot_summary.xlsx` → "Refund-Cancel-Fail by Region") and excluded from completed sales |
| Ship date before order date | Flagged `ship_before_order = TRUE`; record kept, not deleted |

"Final completed sales" throughout all reports = `order_status = "Completed"` **AND** `payment_status = "Paid"`.

---

## 4. Assumptions Made

1. **Valid discount range = 0%–30%.** The data showed a clean cluster of legitimate-looking discount tiers (0%, 5%, 10%, 15%, 20%, 25%) and a separate cluster of clearly invalid values (55%, 65%, 70%, 85%, and all negatives), with no values in between. 30% was chosen as a safe upper boundary above the highest observed legitimate tier (25%).
2. **Missing discount → 0** was only applied where `quantity` and `unit_price` were both present and valid, per the business rule; this held true for all 18 affected records, so no record was left without a usable `cleaned_discount` due to this rule.
3. **Date parsing order** — when trying multiple date formats, `DD Mon YYYY` was tried first (least ambiguous, contains a month name), then `MM/DD/YYYY`, then `DD-MM-YYYY`, then `YYYY-MM-DD`. No values were ambiguous enough to match more than one format incorrectly, since day values >12 made the `MM/DD` vs `DD-MM` formats self-disambiguating in the slash/dash-separated cases.
4. **Conflicting duplicates were not resolved automatically.** Without knowing which internal source system is authoritative, it was assumed safer to retain and flag all versions rather than guess which one is "correct."
5. **Completed + non-Paid payment status (2 records)** was treated as a data inconsistency. These records remain in the cleaned file with their original status values, but were excluded from completed-sales totals based on `payment_status`, since revenue should only be recognized once payment has actually been received.
6. **Calculation mismatches (sales/profit) were not overwritten.** The original `sales`/`profit` figures are preserved unchanged, and `calculated_sales`/`calculated_profit` are added as new columns, so neither the original system-of-record values nor the recalculated values are lost.

---

## 5. Records Removed

- **20 rows** removed — exact duplicate copies (identical across all original columns). The first occurrence of each was kept.
- No other rows were removed. All rows with missing values, invalid discounts, invalid dates, or conflicting duplicate order_ids were **kept and flagged**, not deleted.

## 6. Records Flagged

- **73 records** flagged `data_quality_flag = Invalid` (unparseable/ship-before-order dates, invalid discounts, or conflicting duplicates).
- **83 records** flagged `data_quality_flag = Warning` (missing-value fills or calculation mismatches).
- **756 records** flagged `data_quality_flag = Clean`.
- Full breakdowns are in `outputs/data_quality_report.xlsx`.

## 7. Limitations of This Cleaning Process

- **Conflicting duplicates remain unresolved.** The 12 conflicting order_ids still have multiple rows in `cleaned_orders.xlsx`. Any sales/profit totals computed without filtering out `conflicting_duplicate = TRUE` (or further investigating which row is correct) will double-count these orders. The pivot summaries in this submission do **not** explicitly exclude them, consistent with not silently deleting flagged data — a business owner should resolve these before using the totals for financial reporting.
- **Invalid discounts were excluded from `calculated_sales`, not corrected.** Because `cleaned_discount` is blank for these 31 rows, `calculated_sales` for those specific rows effectively assumes a 0% discount, since the formula treats a blank discount as 0 for calculation purposes. This understates the true discount impact for those specific rows but was the safest option without knowing the correct intended discount.
- **Ship-before-order records (21 rows) were flagged, not corrected**, since it isn't possible to know whether the order date or ship date was the one entered incorrectly.
- **No external validation** of customer, state, or city data against an authoritative master list was performed (e.g. confirming city-state pairs are real combinations) — values were cleaned for formatting only, not against business-reference data.
- **Discount range thresholds (0–30%) are a data-driven assumption**, not a rule provided directly in the source instructions, and should be confirmed against actual company discount policy if available.
