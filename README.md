# FoodLens — Food Inspections Data Pipeline

An end-to-end, metadata-driven data engineering pipeline built on **Databricks** and **Delta Lake** that ingests food inspection data from Chicago and Dallas public APIs, transforms it through Bronze, Silver, and Gold layers following **Medallion Architecture**, and delivers a Kimball-style star schema for analytics and reporting.

---

## Overview

This project processes **387,000+ food inspection records** from two cities with completely different schemas — Chicago and Dallas — and unifies them into a single, analytics-ready dimensional model. The pipeline handles real-world data engineering challenges including schema mismatches, data quality validation, SCD Type 2 history tracking, and multi-source integration.

---

## Architecture

```
Chicago Open Data API ──┐
                         ├──► Raw Volume (Parquet) ──► Bronze (Delta) ──► Silver (Delta) ──► Gold (Delta)
Dallas Open Data API  ──┘
                                                          ↑
                                              DQX Validation + Business Rules
```

| Layer | Storage | Description |
|---|---|---|
| Raw | Databricks Volume (Parquet) | Exact copy of source CSVs with timestamped paths — never overwritten |
| Bronze | Unity Catalog Delta Tables | Raw data landed as-is with audit metadata columns |
| Silver | Unity Catalog Delta Tables | Cleansed, standardized, deduplicated — one row per violation per inspection |
| Gold | Unity Catalog Delta Tables | Kimball star schema — facts and dimensions for analytics |

---

## Data Sources

| City | Source | Raw Rows |
|---|---|---|
| Chicago | Chicago Open Data Portal | 308,161 |
| Dallas | Dallas Open Data Portal | 78,984 |

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Databricks | Compute, Unity Catalog, Delta table management |
| PySpark | All transformation and validation logic |
| Delta Lake | ACID transactions, SCD Type 2 MERGE operations |
| Unity Catalog | Governance, schema management, volume storage |
| Power BI | Star schema semantic model and dashboards |

---

## Pipeline Notebooks

| Notebook | What It Does |
|---|---|
| `Util.ipynb` | Centralized constants — all catalog, schema, table, and path variables |
| `01_Environment_Setup.ipynb` | Creates catalog, schemas, volumes, metadata tables, and Gold layer DDL |
| `02_Raw_to_Bronze_Load.ipynb` | Reads CSVs from Raw Volume, writes Parquet to Staging Volume, loads Bronze Delta tables, logs to child_metadata |
| `03_Bronze_to_Silver_Load.ipynb` | DQX profiling + validation, Chicago/Dallas transformation, business rules, deduplication, writes Silver |
| `04_Silver_to_Gold_Load.ipynb` | Builds star schema — loads dim_date, dim_violation_detail, dim_restaurant (SCD2), fact_inspections, fact_violations via Delta MERGE |

---

## Key Engineering Challenges

### 1. Schema Mismatch
Chicago and Dallas datasets had completely different formats:
- Chicago stored all violations as a single pipe-delimited string per inspection
- Dallas stored violations across 25 separate wide columns per inspection row

### 2. Multi-Source Harmonization
Both datasets were standardized to a common Silver schema — one row per violation per inspection — using regex parsing for Chicago and a slot-based union pattern for Dallas.

### 3. Surrogate Key Generation
MD5 hashing used to generate consistent business keys across both cities:
- `restaurant_id` — MD5(DBA_Name + facility_type + address + zip)
- `violation_id` — MD5(violation_code + description + detail)
- `inspection_id` (Dallas) — MD5(DBA_Name + inspection_date + address + zip)

---

## Data Quality (DQX)

A custom DQX engine was implemented at the Silver layer with the following validation rules:

| Rule | City | Action |
|---|---|---|
| DBA_Name cannot be null or empty | Both | Drop bad row |
| inspection_date cannot be null | Both | Drop bad row |
| zip_code must match `^\d{5}$` | Both | Drop bad row |
| violation_score must be 0–100 | Dallas | Drop bad row |
| inspection_result cannot be null | Chicago | Drop bad row |
| PASS cannot have Critical/Urgent violation | Both | Drop inspection rows |
| Score >= 90 cannot have > 3 violations | Dallas | Drop inspection rows |
| Duplicate violations deduplicated | Both | Keep distinct |

**DQX Results:** 1,300 bad rows removed from Dallas (1,186 invalid zip codes, 75 null DBA names, 39 out-of-range scores). All post-write integrity checks passed.

---

## Dimensional Model

![Dimensional Model](Dashboard%20Images/dimensional-model.png)

| Table | Type | Grain | Rows |
|---|---|---|---|
| `dim_date` | Dimension | One row per unique inspection date | 4,518 |
| `dim_restaurant` | SCD Type 2 | One row per version of a restaurant entity | 43,677 |
| `dim_violation_detail` | Dimension | One row per unique violation code/description | 911 |
| `fact_inspections` | Fact | One row per inspection event | 279,471 |
| `fact_violations` | Fact | One row per violation per inspection | 1,179,288 |

### SCD Type 2 — dim_restaurant
`dim_restaurant` tracks restaurant history using the UNION ALL NULL-key MERGE pattern:
- `change_hash` (MD5 of AKA_Name) detects attribute changes
- On change: existing record expired (`is_current=false`, `effective_end_date=current_timestamp`)
- New record inserted (`effective_end_date=9999-12-31`, `is_current=true`)

---

## Dashboards

Four Power BI dashboards built on the Gold layer star schema.

### Overview
![Overview Dashboard](Dashboard%20Images/Dashboard-1.png)

### Violations Analysis
![Violations Dashboard](Dashboard%20Images/Dashboard-2.png)

### Restaurant Analysis
![Restaurant Dashboard](Dashboard%20Images/Dashboard-3.png)

### Inspection and Validation Report
![Inspection Report](Dashboard%20Images/Dashboard-4.png)

---

## How to Run

### Prerequisites
- Databricks workspace with Unity Catalog enabled
- Cluster with internet access (for API downloads)

### Step 1 — Configure Util notebook
Update constants in `Notebooks/Util.ipynb`:
```python
CATALOG       = "foodlens"
BRONZE_SCHEMA = "bronze_zone"
SILVER_SCHEMA = "silver_zone"
GOLD_SCHEMA   = "gold_zone"
META_SCHEMA   = "pipeline_metadata"
```

### Step 2 — Upload source CSVs to Volume
Upload both CSV files to:
```
/Volumes/foodlens/bronze_zone/vol_raw_files/
```

### Step 3 — Run notebooks in order
```
01_Environment_Setup     → Creates all infrastructure
02_Raw_to_Bronze_Load    → Ingests raw CSVs to Bronze
03_Bronze_to_Silver_Load → Cleanses and validates to Silver
04_Silver_to_Gold_Load   → Builds star schema in Gold
```

### Step 4 — Monitor pipeline runs
```sql
SELECT * FROM foodlens.pipeline_metadata.child_metadata
ORDER BY execution_time DESC;
```

---

## Repository Structure

```
FoodLens-Databricks-Pipeline/
├── Notebooks/
│   ├── Util.ipynb
│   ├── 01_Environment_Setup.ipynb
│   ├── 02_Raw_to_Bronze_Load.ipynb
│   ├── 03_Bronze_to_Silver_Load.ipynb
│   └── 04_Silver_to_Gold_Load.ipynb
├── Dashboard Images/
│   ├── Dashboard-1.png
│   ├── Dashboard-2.png
│   ├── Dashboard-3.png
│   ├── Dashboard-4.png
│   └── dimensional-model.png
├── Food_Inspections_Report.pdf
├── Source_to_Target_Mapping_FoodLens.xlsx
└── README.md
```

---

## Good Practices

- **Metadata-driven pipeline** — parent_metadata drives active source selection; child_metadata logs every run
- **Idempotent loads** — all Gold tables use Delta MERGE, safe to rerun without duplication
- **Audit columns** — `_date_to_warehouse`, `_batch_id`, `_ingested_at`, `_bronze_loaded_at` at every layer
- **Quarantine pattern** — invalid Gold-layer records routed to `Quarantine_Inspection` table with reason codes
- **Parameterized notebooks** — no hardcoded values; all constants centralized in `Util.ipynb`
- **Row-count reconciliation** — source vs. target counts verified at every Bronze load

---

## Data Sources

- Chicago: [Chicago Open Data Portal](https://data.cityofchicago.org/Health-Human-Services/Food-Inspections/4ijn-s7e5)
- Dallas: [Dallas Open Data Portal](https://www.dallasopendata.com/Services/Restaurant-and-Food-Establishment-Inspections-Octo/dri5-wcct)
