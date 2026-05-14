# Cleaning Log — Gallagher Sales Pipeline

## Overview

All cleaning performed in **Power Query (M language)**. The source Excel file was never modified.
Every action is recorded in the Applied Steps panel — the full audit trail exists inside the .pbix file.

---

## Issues Found During Profiling

Column Distribution and Column Statistics in Power Query were used to profile every column before any transformation began.

| # | Column | Issue Found | Rows Affected | Severity |
|---|--------|-------------|--------------|----------|
| 1 | Date | abc/123 mixed type — invalid text values ("2", "kj", "ioo") mixed with valid DD/MM/YYYY dates | 3 invalid | High |
| 2 | Expected Decision Date | abc/123 mixed type — "=" formula artefacts from Excel | 2 invalid | High |
| 3 | Annual Revenue | 6 error-state cells (#VALUE!) from broken Excel formulas | 6 | High |
| 4 | Niche Affiliations | 1 error-state #NAME? cell, ~18 blank/null values | ~19 | Medium |
| 5 | Stage Name | Long inconsistent strings — exact-match dependent | 87 | Medium |
| 6 | Opportunity Name | Mixed case values | ~15 | Low |
| 7 | All text columns | Hidden spaces, non-printable characters | Unknown | Medium |
| 8 | Whole table | Duplicate rows — exact and near-duplicate | 93 | High |

---

## Cleaning Steps Applied

### Step 1 — Replace Errors on Niche Affiliations (MUST come before Trim)

- **Action:** Transform tab → Replace Errors → type "Other" → OK
- **Why first:** `#NAME?` is an error-state cell — not a text value. Trim cannot operate on error cells. Filtering this column before handling the error causes a `DataFormat.Error` crash on the entire table.
- **Key principle:** Error-state cells and text cells are different things in Power Query. Handle errors before applying text transformations.

---

### Step 2 — Trim and Clean all text columns

- **Action:** Select columns: Primary Producer, Stage Name, Account Name, Opportunity Name, Niche Affiliations → Transform → Format → Trim. Repeat with Format → Clean.
- **Why:** Trim removes leading/trailing and double-spaces. Clean removes invisible non-printable characters. Both are required before any Replace Values step — a hidden space around a value like `" Stage 1"` prevents exact-match replacement from finding it.

---

### Step 3 — Fix Annual Revenue

- **Action:** Replace Errors → null. Change type → Fixed Decimal Number.
- **Why null and not 0:** The 6 error rows had broken Excel formulas — the revenue was not recorded, not genuinely zero. Using 0 would reduce `AVERAGEX` results. Example: avg(10000, 0, 5000) = 5000 vs avg(10000, null, 5000) = 7500. Null is honest.
- **Why Fixed Decimal:** Appropriate for financial currency values — avoids floating point rounding errors.
- **No text conversion required:** The errors were error-state cells, not text. `Replace Errors` operates directly without needing to force the column to Text type first.

---

### Step 4 — Fix Date column (5-step approach)

1. Force column to Text type
2. Trim + Clean (removes any residual spaces around values)
3. Replace known invalid values: `"2"` → null, `"kj"` → null, `"ioo"` → null
4. Change Type → Using Locale → Data Type: Date → Locale: English (United Kingdom)
5. Replace Errors → null

**Why force to Text first:** Prevents the numeric value `"2"` from being interpreted as Excel serial number 2 (= Jan 2, 1900) before replacement.

**Why UK Locale:** Source dates are DD/MM/YYYY. Power BI defaults to US MM/DD/YYYY — any date with day > 12 would error without locale specification.

---

### Step 5 — Fix Expected Decision Date

Same 5-step approach as Step 4.
- The `"="` value is an Excel formula artefact. Replaced with null in Step 3 before locale conversion.

---

### Step 6 — Fix Niche Affiliations remaining nulls and blanks

- **Action:** Replace Values → find null → replace with "Other". Replace Values → find empty string → replace with "Other".
- **Result:** No blanks, no nulls, no error values. All uncategorised niches merged into the existing "Other" category rather than creating a new "Unclassified" label.

---

### Step 7 — Standardise Stage Name (dynamic extraction)

- **Action:** Add Column → Custom Column
```m
Stage_ID = Number.FromText(Text.Trim(Text.BeforeDelimiter([Stage Name], "-")))
```
- This extracts the number before the first hyphen ("1", "2", "3") and converts it to a Whole Number integer.
- The original Stage Name text column was removed from the fact table.
- Stage labels were separated into a dedicated **Dim_Stage** dimension table (created manually via Enter Data). This gives sort control for the funnel chart and follows proper dimensional modelling.

**Why dynamic extraction instead of Replace Values:**
Three hardcoded `Replace Values` steps break if there is a hidden space in the string or if a description changes. `Text.BeforeDelimiter` extracts the number regardless of what the rest of the string says — scalable to future stages without changing the transformation logic.

---

### Step 8 — Fix Opportunity Name

- **Action:** Trim + Clean only. No Capitalize Each Word applied.
- **Why no capitalisation:** Opportunity Name is a free-text field containing business acronyms (CBCP, IASME, UIB). Capitalize Each Word would distort these. Trim is sufficient for a display-only field not used in grouping or joining.

---

### Step 9 — Two-stage deduplication

**Stage A — Exact match on all columns**
- Home → Remove Rows → Remove Duplicates
- Removes rows that became identical after Steps 3–8 resolved mixed revenue and date formatting
- Result: 180 → 93 rows

**Stage B — Business-key deduplication**
- Removed remaining duplicates using Account Name + Opportunity Name as composite key
- **Reasoning:** Stage, Revenue, and Date are mutable attributes of an opportunity — they can legitimately differ across records of the same deal at different points in time. Account Name + Opportunity Name together define the unique business event.
- Result: 93 → 87 rows

| After Stage A | After Stage B |
|--------------|--------------|
| 93 rows | 87 rows |

> Note: 3 rows contain null dates. These rows were **retained** in the model because the opportunity and revenue data are still valid. They are excluded only from time-intelligence calculations using a separate `Total Revenue (Valid Dates)` base measure. The final model contains 87 rows.

---

### Step 10 — Add Opportunity_ID surrogate key

- **Action:** Add Column → Index Column → From 1 → rename to Opportunity_ID → move to first column position → change type to Whole Number.
- **Critical rule:** Always add after deduplication. Adding an index before deduplication makes every row unique by definition — Power Query finds zero duplicates.

---

### Step 11 — Add Is_Closed_Won binary flag

- **Action:** Add Column → Custom Column
```m
= if [Stage_ID] = 3 then 1 else 0
```
- Change type to Whole Number.
- **Why integer flag:** Simplifies DAX — `Win Rate % = DIVIDE([Closed Won Count], [Total Opportunities], 0)` without needing text-based filters.
- **Why Stage_ID = 3 (not Stage Name text):** By this point the fact table uses integer foreign keys. Stage_ID = 3 corresponds to "Stage 3 - Closed Won" per the Dim_Stage mapping.

---

### Step 12 — Integer foreign key merges

**Merge 1 — Producer_ID**
- Added Index Column (1–5) to Dim_Producers → named Producer_ID
- Merged Fact_Opportunities ← Dim_Producers on Primary Producer (Left Outer Join)
- Expanded merge: brought in Producer_ID only
- Removed Primary Producer text column from fact table

**Merge 2 — Account_ID**
- Merged Fact_Opportunities ← Dim_Accounts on Account Name (Left Outer Join)
- Expanded merge: brought in Acct_ID only → renamed to Account_ID
- Removed Account Name text column from fact table

**Why integer/ID foreign keys:**
Text joins risk silent mismatches if any whitespace difference exists between tables. Integer joins are deterministic. Removing the text columns from the fact table keeps it lean — only measures and foreign keys.

---

## Final Dataset Statistics

| Metric | Value |
|--------|-------|
| Final row count | 87 |
| Rows with valid dates | 84 |
| Rows with null dates | 3 (retained in model) |
| Errors in any column | 0 |
| Null revenue rows | 6 (null preserved, not 0) |
| Tables in model | 5 (Fact + Dim_Accounts + Dim_Producers + Dim_Stage + Calendar) |
| Relationships | 4 (all Many-to-One, Single cross-filter direction) |
