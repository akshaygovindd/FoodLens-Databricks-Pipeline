# FoodLens — Food Inspections Data Pipeline

An end-to-end data engineering pipeline built on Databricks and Delta Lake that ingests food inspection records from Chicago and Dallas, processes them through Bronze, Silver, and Gold layers, and loads a star schema for analytics and reporting.

The pipeline processes 387,000+ records from two cities that have nothing in common structurally. Chicago stores all violations as a single pipe-delimited string. Dallas spreads them across 25 separate columns. Getting them into one clean, unified model is the whole challenge.

## Architecture

```
Chicago Open Data API ──┐
                         ├──► Raw Volume (Parquet) ──► Bronze (Delta) ──► Silver (Delta) ──► Gold (Delta)
Dallas Open Data API  ──┘
                                                          ↑
                                              DQX Validation + Business Rules
```

| Layer | Storage | What it does |
|---|---|---|
| Raw | Databricks Volume | Parquet files with timestamped paths. Never overwritten. |
| Bronze | Delta Tables | Raw data landed as-is with audit metadata columns |
| Silver | Delta Tables | Cleaned, standardized, deduplicated. One row per violation per inspection. |
| Gold | Delta Tables | Kimball star schema built for analytics |

## Data Sources

| City | Source | Raw Rows |
|---|---|---|
| Chicago | Chicago Open Data Portal | 308,161 |
| Dallas | Dallas Open Data Portal | 78,984 |

## Tech Stack

| Tool | Purpose |
|---|---|
| Databricks | Compute, Unity Catalog, Delta table management |
| PySpark | All transformation and validation logic |
| Delta Lake | ACID transactions, SCD Type 2 MERGE operations |
| Unity Catalog | Governance, schema management, volume storage |
| Power BI | Star schema semantic model and dashboards |

## Pipeline Notebooks

| Notebook | What it does |
|---|---|
| `Util.ipynb` | All catalog, schema, table, and path constants in one place |
| `01_Environment_Setup.ipynb` | Creates catalog, schemas, volumes, metadata tables, and Gold layer DDL |
| `02_Raw_to_Bronze_Load.ipynb` | Reads CSVs, writes timestamped Parquet to staging, loads Bronze Delta tables, logs to child_metadata |
| `03_Bronze_to_Silver_Load.ipynb` | DQX validation, Chicago and Dallas transformation, business rules, deduplication, writes Silver |
| `04_Silver_to_Gold_Load.ipynb` | Loads dim_date, dim_violation_detail, dim_restaurant (SCD2), fact_inspections, fact_violations via Delta MERGE |

## What Made This Hard

**Schema mismatch.** Chicago stores all violations as one pipe-delimited string per inspection. Dallas stores them across 25 separate wide columns. Both had to end up as one row per violation before anything else could happen.

**Surrogate keys across two cities.** No shared identifiers. MD5 hashing on a combination of business fields was the only way to generate consistent keys across both sources.

- `restaurant_id` = MD5(DBA_Name + facility_type + address + zip)
- `violation_id` = MD5(violation_code + description + detail)
- `inspection_id` for Dallas = MD5(DBA_Name + inspection_date + address + zip)

**Dallas has no facility type, no risk category, no inspection ID.** All three had to be derived. Facility type via keyword matching on restaurant name, risk set to Unknown, inspection ID via MD5 hash.

## Data Quality

Custom DQX engine at the Silver layer. Bad rows get dropped before anything reaches Gold.

| Rule | City | Action |
|---|---|---|
| DBA_Name cannot be null or empty | Both | Drop row |
| inspection_date cannot be null | Both | Drop row |
| zip_code must match `^\d{5}$` | Both | Drop row |
| violation_score must be 0 to 100 | Dallas | Drop row |
| inspection_result cannot be null | Chicago | Drop row |
| PASS cannot have a Critical or Urgent violation | Both | Drop inspection |
| Score >= 90 cannot have more than 3 violations | Dallas | Drop inspection |
| Duplicate violations | Both | Keep distinct |

Dallas had 1,300 bad rows removed (1,186 invalid zip codes, 75 null DBA names, 39 out-of-range scores). All post-write integrity checks passed.

## Dimensional Model

![Dimensional Model](Dashboard%20Images/dimensional-model.png)

| Table | Type | Grain | Rows |
|---|---|---|---|
| `dim_date` | Dimension | One row per unique inspection date | 4,518 |
| `dim_restaurant` | SCD Type 2 | One row per version of a restaurant entity | 43,677 |
| `dim_violation_detail` | Dimension | One row per unique violation code and description | 911 |
| `fact_inspections` | Fact | One row per inspection event | 279,471 |
| `fact_violations` | Fact | One row per violation per inspection | 1,179,288 |

`dim_restaurant` uses the UNION ALL NULL-key MERGE pattern for SCD Type 2. `change_hash` is MD5 of AKA_Name. When it changes, the old record gets expired and a new one is inserted with `effective_end_date = 9999-12-31`.

## Dashboards

### Overview
![Overview Dashboard](Dashboard%20Images/Dashboard-1.png)

### Violations Analysis
![Violations Dashboard](Dashboard%20Images/Dashboard-2.png)

### Restaurant Analysis
![Restaurant Dashboard](Dashboard%20Images/Dashboard-3.png)

### Inspection and Validation Report
![Inspection Report](Dashboard%20Images/Dashboard-4.png)

## How to Run

**Prerequisites:** Databricks workspace with Unity Catalog enabled and a cluster with internet access.

Update the constants in `Notebooks/Util.ipynb` to match your workspace:

```python
CATALOG       = "foodlens"
BRONZE_SCHEMA = "bronze_zone"
SILVER_SCHEMA = "silver_zone"
GOLD_SCHEMA   = "gold_zone"
META_SCHEMA   = "pipeline_metadata"
```

Upload both source CSV files to your Raw Volume:

```
/Volumes/foodlens/bronze_zone/vol_raw_files/
```

Run the notebooks in order:

```
01_Environment_Setup       Creates all infrastructure
02_Raw_to_Bronze_Load      Ingests raw CSVs to Bronze
03_Bronze_to_Silver_Load   Cleanses and validates to Silver
04_Silver_to_Gold_Load     Builds star schema in Gold
```

To check pipeline run history:

```sql
SELECT * FROM foodlens.pipeline_metadata.child_metadata
ORDER BY execution_time DESC;
```

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
├── Food_Inspections_Chicago_Dallas_Requirements.pdf
├── Source_to_Target_Mapping_FoodLens.xlsx
└── README.md
```

## Good Practices

Every run is logged. Every load is idempotent. Nothing is hardcoded.

- `parent_metadata` drives which sources run. `child_metadata` logs status, row counts, and batch IDs for every execution.
- All Gold tables use Delta MERGE instead of overwrite so reruns never duplicate data.
- Audit columns (`_date_to_warehouse`, `_batch_id`, `_ingested_at`, `_bronze_loaded_at`) at every layer.
- Invalid Gold-layer records go to a `Quarantine_Inspection` table with reason codes instead of silently failing.
- Source vs. target row counts are reconciled at every Bronze load before anything moves forward.

## Data Sources

- Chicago: [Chicago Open Data Portal](https://data.cityofchicago.org/Health-Human-Services/Food-Inspections/4ijn-s7e5)
- Dallas: [Dallas Open Data Portal](https://www.dallasopendata.com/Services/Restaurant-and-Food-Establishment-Inspections-Octo/dri5-wcct)
