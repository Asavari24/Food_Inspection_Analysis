# Food_Inspection_Analysis

# Food Inspection Analysis — End-to-End Data Engineering Pipeline

## Project Overview

An end-to-end data engineering pipeline analyzing food inspection data from **Chicago (~308K rows, 17 columns)** and **Dallas (~79K rows, 114 columns)** using Databricks, Delta Lake, and Power BI. The project follows the **Medallion Architecture (Bronze → Silver → Gold)** with Kimball-style dimensional modeling, DQX data quality validation, SCD Type 2 change tracking, and automated pipeline orchestration.

**Course:** DAMG 7370 — Designing Advanced Data Architectures for Business Intelligence  
**University:** Northeastern University  
**Term:** Spring 2026

## Architecture

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Raw (CSV)  │ →  │   Bronze    │ →  │   Silver    │ →  │    Gold     │ → Power BI
│  Volume     │    │  Delta Lake │    │  Cleansed   │    │ Star Schema │   Dashboard
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                   01_Raw_Ingestion   03_Bronze_to_      04_Silver_to_
                                      Silver             Gold
```

| Layer | Purpose | NULL Handling |
|-------|---------|---------------|
| **Raw** | Source CSVs in Databricks Volume — untouched | As-is |
| **Bronze** | Schema-applied Delta tables, column renames, MERGE-based ingestion | As-is |
| **Silver** | DQX-validated, bad rows quarantined, violations parsed/standardized | NULLs preserved (Kimball) |
| **Gold** | Star schema dimensional model, SCD2, sentinel values | `"Unknown"` for strings, `-9999` for integers, `-9999.0` for doubles |

## Data Sources

| Dataset | Source | Rows | Columns |
|---------|--------|------|---------|
| Chicago Food Inspections | [Chicago Data Portal](https://data.cityofchicago.org/) | ~308,000 | 17 |
| Dallas Food Inspections | [Dallas OpenData](https://www.dallasopendata.com/) | ~79,000 | 114 (25 repeating violation groups) |

## Dimensional Model (Gold Layer)

Star schema with `fact_inspection` at the center, connected to 6 dimensions and a bridge table:

| Table | Type | Key | Notes |
|-------|------|-----|-------|
| `fact_inspection` | Fact | INSPECTION_SK | Append mode with dedup (preserves SCD2 history) |
| `dim_date` | Dimension (Fixed) | DATE_SK | Calendar attributes derived from inspection dates |
| `dim_inspection_type` | Dimension (Fixed) | INSPECTION_TYPE_SK | Per city |
| `dim_inspection_result` | Dimension (Fixed) | INSPECTION_RESULT_SK | Chicago: direct from Results; Dallas: derived from score |
| `dim_location` | Dimension (Fixed) | LOCATION_SK | Address, City, State, Zip, Lat/Long with coordinate validation |
| `dim_violation` | Dimension (Fixed) | VIOLATION_SK | Both cities unified, codes standardized |
| `dim_establishment` | Dimension (SCD Type 2) | ESTABLISHMENT_SK | MD5 hash change detection, EFF_START_DATE/EFF_END_DATE tracking |
| `bridge_inspection_violation` | Bridge | VIOLATION_SK + INSPECTION_SK | Many-to-many link between inspections and violations |

## Pipeline Notebooks

| # | Notebook | Description |
|---|----------|-------------|
| 01 | `01_Raw_Ingestion` | Volume CSV → Bronze Delta. MERGE-based ingestion (preserves changes for SCD2) |
| 02 | `Chicago_Food_Inspections_Data_Profiling_DQX` | DQX profiling on Chicago Bronze data. Parses pipe-delimited violations |
| 03 | `Dallas_Food_Inspections_Data_Profiling_DQX` | DQX profiling on Dallas Bronze data. Explodes 25 violation column groups |
| 04 | `03_Bronze_to_Silver_FoodInspection` | Bronze → Silver with DQX validation, quarantine, violation standardization |
| 05 | `04_Silver_to_Gold_FoodInspection` | Silver → Gold dimensional model with SCD2 and Kimball sentinel values |
| 06 | `05_SCD2_Validation_Test` | SCD2 test notebook — validates insert, update, and history tracking |

## Data Validation Rules (Silver Layer — DQX)

| # | Rule | Cities |
|---|------|--------|
| 1 | Restaurant Name, Inspection Date & Type, Zip cannot be NULL | Both |
| 2 | Zip codes must be valid format (5-digit or 5+4) | Both |
| 3 | Violation score cannot exceed 100 | Dallas |
| 4 | Inspection Results cannot be NULL | Chicago |
| 5 | Every inspection must have ≥ 1 unique violation; duplicates loaded as distinct | Both |
| 6 | Score ≥ 90 → max 3 violations | Dallas |
| 7 | No PASS if any violation contains Urgent/Critical terms | Dallas |
| 8 | Derive violation score from Results (Pass=90, Pass w/ Conditions=80, Fail=70, No Entry=0) | Chicago |

## Violation Schema Standardization

| Field | Chicago Source | Dallas Source | Standardized Output |
|-------|--------------|--------------|---------------------|
| Violation_Code | Extracted from `53. DESCRIPTION` via regex | Extracted from `*43 DESCRIPTION` via regex | Integer (NULL → -9999 in Gold) |
| Violation_Description | Text between `CODE.` and `- Comments:` | Text after stripping `*CODE ` prefix | Clean string |
| Violation_Comments | Text after `- Comments:` | From `Violation_Memo_N` column | String (NULL preserved in Silver) |

## SCD Type 2 Implementation

Implemented on `dim_establishment` using MD5 hash-based change detection:

| Operation | What Happens |
|-----------|-------------|
| **Initial Load** | All records inserted with `IS_CURRENT='Y'`, `EFF_END_DATE=NULL` |
| **Update Detected** | Old row: `IS_CURRENT='N'`, `EFF_END_DATE=today`. New row: `IS_CURRENT='Y'`, `EFF_START_DATE=today` |
| **New Record** | Inserted with `IS_CURRENT='Y'`, `EFF_END_DATE=NULL` |
| **No Changes** | Nothing happens — idempotent |

Hash columns: `DBA_NAME, AKA_NAME, LICENSE_NUMBER, FACILITY_TYPE, RISK_CATEGORY`

## Pipeline Orchestration (Databricks Jobs)

### Job 1: `Food_Inspection_Pipeline` (Full Load)
```
raw_ingestion → bronze_to_silver → silver_to_gold
```

### Job 2: `Food_Inspection_SCD2_Test` (Incremental / SCD2 Testing)
```
bronze_to_silver → silver_to_gold
```

Job 1 runs the complete pipeline including MERGE-based raw ingestion. Job 2 skips raw ingestion for testing SCD2 changes made directly to Bronze.

## Gold Layer Data Quality

All Gold dimension tables include a default "Unknown" row (SK = -9999) per Kimball. All fact table foreign keys use `coalesce(FK, -9999)` to ensure every fact row joins to a dimension.

| Sentinel | Data Type | Value |
|----------|-----------|-------|
| String | `"Unknown"` | For missing text fields |
| Integer | `-9999` | For missing numeric fields |
| Double | `-9999.0` | For missing coordinates |
| Date | `NULL` | For `EFF_END_DATE` on current SCD2 records |

## Power BI Dashboard

Dashboard pages examining food inspection results by:

- **Inspection Result** — Pass/Fail/Pass w/ Conditions breakdown
- **Inspection Type** — Canvass, License, Complaint, etc.
- **Risk Category** — High/Medium/Low distribution
- **Facility Type** — Restaurant, Grocery Store, School, etc.
- **Violations** — Codes, descriptions, points
- **Business Inspected** — DBA, AKA, License lookup
- **Location** — Map visualization with Lat/Long (filtered to valid US coordinates)
- **Dashboard** — Summary overview

Data source: Gold layer CSV exports loaded in Import mode with star schema relationships.

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Processing | Databricks (PySpark, Delta Lake, Spark SQL) |
| Data Quality | Databricks Labs DQX |
| Storage | Unity Catalog (`chicago_dallas_food_inspection`) |
| Dimensional Modeling | Navicat (ER diagrams) |
| Visualization | Power BI Desktop (Import mode) |
| Orchestration | Databricks Jobs (DAG-based) |
| Version Control | Git / GitHub |

## Catalog Structure

```
chicago_dallas_food_inspection/
├── bronze/
│   ├── chicago_raw
│   └── dallas_raw
├── silver/
│   ├── chicago_inspections
│   ├── dallas_inspections
│   ├── chicago_violations
│   ├── dallas_violations
│   ├── chicago_quarantine
│   └── dallas_quarantine
├── gold/
│   ├── dim_date
│   ├── dim_inspection_type
│   ├── dim_inspection_result
│   ├── dim_location
│   ├── dim_violation
│   ├── dim_establishment (SCD2)
│   ├── fact_inspection
│   └── bridge_inspection_violation
├── chicago/    (profiling outputs)
└── dallas/     (profiling outputs)
```

## Repository Structure

```
Food_Inspection_Analysis/
├── README.md
├── notebooks/
│   ├── 01_Raw_Ingestion.ipynb
│   ├── Chicago_Food_Inspections_Data_Profiling_DQX.ipynb
│   ├── Dallas_Food_Inspections_Data_Profiling_DQX.ipynb
│   ├── 03_Bronze_to_Silver_FoodInspection.ipynb
│   ├── 04_Silver_to_Gold_FoodInspection.ipynb
│   └── 05_SCD2_Validation_Test.ipynb
├── docs/
│   ├── Food_Inspection_Mapping_Doc.xlsx
│   └── ER_Diagram.png
└── dashboard/
    └── Food_Inspection_Dashboard.pbix
```

## Known Limitations

- **Dallas has no Inspection_ID**: Bridge table uses `-9999` for `INSPECTION_SK` on Dallas rows (305K rows). Violation details still accessible via `VIOLATION_SK → dim_violation`.
- **Dallas lacks Facility_Type, License_Number, Risk_Category**: These fields don't exist in the Dallas source. Gold correctly replaces with `"Unknown"` sentinel values.
- **Static source CSVs**: `01_Raw_Ingestion` uses MERGE to prevent overwriting Bronze changes. In production, source data would arrive incrementally.

## How to Run

1. Upload source CSVs to Databricks Volume: `/Volumes/chicago_dallas_food_inspection/raw_data/food_inspection/`
2. Import all notebooks to Databricks Workspace
3. Run `Food_Inspection_Pipeline` job for initial load
4. For SCD2 testing: modify Bronze data, then run `Food_Inspection_SCD2_Test` job
5. Export Gold tables as CSV and load into Power BI
