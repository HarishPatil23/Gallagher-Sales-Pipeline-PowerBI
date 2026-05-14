# Data Dictionary — Gallagher Sales Pipeline

## Model Overview

| Component | Detail |
|-----------|--------|
| Total tables | 5 (1 Fact + 3 Dimensions + 1 Calendar) |
| Total rows (fact) | 87 |
| Rows with valid dates | 84 |
| Null-date rows | 3 (retained — excluded from time intelligence only) |
| Relationships | 4 (all Many-to-One, Single cross-filter direction) |

---

## Fact_Opportunities

| Column | Data Type | Description | Cleaning / Notes |
|--------|-----------|-------------|-----------------|
| Opportunity_ID | Whole Number | Surrogate primary key (1–87). Added via Index Column after deduplication. | Generated in Power Query — Index Column → From 1 |
| Date | Date | Date the opportunity was first created / initial contact made | Force Text → Replace invalids → Change Type (UK Locale) → Replace Errors → null. 3 rows remain null — retained. |
| Producer_ID | Whole Number | Foreign key → Dim_Producers[Producer_ID] | Brought in via Power Query Merge on Primary Producer. Original text column removed. |
| Stage_ID | Whole Number | Foreign key → Dim_Stage[StageID]. Values: 1, 2, or 3 only | Extracted via: `Number.FromText(Text.Trim(Text.BeforeDelimiter([Stage Name], "-")))` |
| Account_ID | Text | Foreign key → Dim_Accounts[Acct_ID]. Format: "A101", "A102" etc. | Brought in via Power Query Merge on Account Name. Original text column removed. |
| Opportunity_Name | Text | Free-text description of the specific opportunity | Trim + Clean only. Capitalize Each Word intentionally NOT applied — preserves business acronyms (IASME, CBCP, UIB). |
| Niche_Affiliations | Text | Industry sector / niche classification | Replace Errors → "Other" (done before Trim). Then nulls/blanks → "Other". |
| Expected_Decision_Date | Date | Anticipated date for client decision | Same 5-step approach as Date column. |
| Annual_Revenue | Fixed Decimal Number | Expected annual revenue (£ GBP) if the deal is won | Replace Errors → null. Not 0. Null = revenue not recorded; 0 would distort AVERAGEX calculations. |
| Is_Closed_Won | Whole Number | Binary flag — 1 = Closed Won (Stage 3), 0 = in progress | Custom Column: `if [Stage_ID] = 3 then 1 else 0` |

---

## Dim_Producers

| Column | Data Type | Description | Notes |
|--------|-----------|-------------|-------|
| Producer_ID | Whole Number | Primary key (1–5). Created via Add Column → Index Column → From 1. | Used as integer foreign key in Fact |
| Primary_Producer | Text | Producer name (Producer1 through Producer5) | Trim + Clean applied |
| Office | Text | Office location (Office 1, Office 2, Office 3) | Trim applied |

---

## Dim_Accounts

| Column | Data Type | Description | Notes |
|--------|-----------|-------------|-------|
| Acct_ID | Text | Primary key. Natural identifier — format "A101", "A102" etc. | Used as foreign key in Fact (Account_ID → Acct_ID) |
| Account_Name | Text | Full client / account name | Trim + Clean applied |
| Primary_Office | Text | Office managing this account | Trim applied |

> **Note on relationship type:** Acct_ID is text format ("A101"), not a pure integer. The Fact[Account_ID] → Dim_Accounts[Acct_ID] relationship is therefore Text-to-Text. This is correct and works reliably — both sides are trimmed to ensure no whitespace mismatch.

---

## Dim_Stage

Created manually in Power Query via Enter Data.

| Column | Data Type | Description | Notes |
|--------|-----------|-------------|-------|
| StageID | Whole Number | Primary key: 1, 2, or 3 | Integer — drives Sort By Column on Stage Name |
| Stage_Name | Text | Clean stage label | Sort By Column = StageID. Ensures correct funnel sort order (1 → 2 → 3). |

| StageID | Stage_Name |
|---------|-----------|
| 1 | Stage 1 - Initial Meeting |
| 2 | Stage 2 - Presentation |
| 3 | Stage 3 - Closed Won |

---

## Calendar

Created as a DAX calculated table. Marked as Date Table.

```dax
Calendar =
ADDCOLUMNS(
    CALENDAR(DATE(2017, 1, 1), DATE(2019, 12, 31)),
    "Year",         YEAR([Date]),
    "Month_Number", MONTH([Date]),
    "Month_Name",   FORMAT([Date], "MMMM"),
    "Quarter",      "Q" & QUARTER([Date]),
    "Year_Month",   FORMAT([Date], "YYYY-MM")
)
```

> **Why hardcoded dates instead of MIN/MAX?**
> Using `MIN(Fact_Opportunities[Date])` caused a blank year issue because 3 rows have null dates — null propagated into the MIN calculation. Hardcoding `DATE(2017,1,1)` to `DATE(2019,12,31)` ensures the Calendar spans the full known data range reliably regardless of null values in the fact table.

| Column | Data Type | Description | Sort By |
|--------|-----------|-------------|---------|
| Date | Date | Primary key — one row per calendar date | — |
| Year | Whole Number | Calendar year | — |
| Month_Number | Whole Number | Month number (1–12) | — |
| Month_Name | Text | Full month name (January, February…) | Month_Number |
| Quarter | Text | Quarter label (Q1, Q2, Q3, Q4) | — |
| Year_Month | Text | Formatted as "YYYY-MM" for chronological axis labels | — |

> **Sort By Column applied:** Month_Name sorted by Month_Number. Prevents alphabetical sorting in visuals (April before August instead of correct January → December order).

---

## Relationships Summary

| From | To | From Column | To Column | Type | Direction |
|------|----|-------------|-----------|------|-----------|
| Fact_Opportunities | Calendar | Date | Date | Many-to-One | Single |
| Fact_Opportunities | Dim_Producers | Producer_ID | Producer_ID | Many-to-One | Single |
| Fact_Opportunities | Dim_Accounts | Account_ID | Acct_ID | Many-to-One (Text) | Single |
| Fact_Opportunities | Dim_Stage | Stage_ID | StageID | Many-to-One (Integer) | Single |

> **All cross-filter directions are Single.** Bidirectional filtering in a star schema causes ambiguous DAX results and unexpected filter propagation between dimension tables.

> **No inactive relationship** between Calendar[Date] and Fact_Opportunities[Expected_Decision_Date] was created. It was not required for any measure in this report.

---

## DAX Measures Reference

All measures stored in a dedicated `_Measures` table.

| Measure | Formula pattern | Used in |
|---------|----------------|---------|
| Total Revenue | SUM | All pages |
| Total Opportunities | COUNTROWS | All pages |
| Closed Won Count | CALCULATE + COUNTROWS + 0 | Page 2, Page 1 (via Win Rate) |
| Closed Won Revenue | CALCULATE + Total Revenue | Page 2 |
| Pipeline Value | CALCULATE + Total Revenue | Page 1 |
| Win Rate % | DIVIDE | Page 1, Page 2 |
| Avg Deal Size | DIVIDE (Total Revenue / Total Opportunities) | Page 1, Page 2 |
| Days to Close | Calculated column — DATEDIFF with negative-day guard | Used by Avg Days to Close |
| Avg Days to Close | ROUND + AVERAGE of Days to Close column | Page 2 |
| Total Revenue (Valid Dates) | CALCULATE + NOT ISBLANK(Date) | Base for time intelligence only |
| Revenue YTD | TOTALYTD | Page 3 |
| Revenue PY | CALCULATE + SAMEPERIODLASTYEAR | Page 3 |
| Revenue Last 3 Months | CALCULATE + DATESINPERIOD | Page 3 |
| YoY Growth % | DIVIDE (Valid Dates - PY) / PY | Page 2 |

**Total: 13 measures + 1 calculated column (Days to Close)**

---

## Data Quality Summary

| Metric | Raw File | After Cleaning |
|--------|----------|----------------|
| Total rows | 180 | 87 |
| Rows with valid dates | — | 84 |
| Rows with null dates | — | 3 (retained) |
| Errors in any column | Multiple | 0 |
| Null revenue rows | 6 | 6 (null preserved, not forced to 0) |
| Duplicate rows removed | 96 | 0 |
| Tables in model | 1 flat sheet | 5 (Fact + 3 Dims + Calendar) |
