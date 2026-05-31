# ECOMMERCE MEDALLION LAKEHOUSE

A data engineering project built on **Databricks** using the **Bronze → Silver → Gold** medallion pattern. Raw ecommerce CSVs are ingested, cleaned, and transformed into BI-ready Delta tables through a series of layered PySpark notebooks, with a final denormalized view powering dashboards.

---

## Architecture Overview

```
Raw CSV Files (Volumes)
        ↓
   [ BRONZE ]  — Raw ingestion, no transformations
        ↓
   [ SILVER ]  — Cleaned, typed, deduplicated
        ↓
   [ GOLD ]    — Business-ready dimension & fact tables
        ↓
  Denormalized View → BI / Dashboard
```

> **Note:** Notebooks are executed manually in order. There is no automated orchestration or scheduling — this is a layered transformation project, not an automated pipeline.

---

## Project Structure

```
ecommerce/
├── 1_setup/
│   └── catalog_setup.ipynb          # Create catalog + bronze/silver/gold schemas
├── 2_medallion_processing_dim/
│   ├── 1_dim_bronze.ipynb           # Ingest dimension CSVs → bronze
│   ├── 2_dim_silver.ipynb           # Clean & transform dimensions → silver
│   └── 3_dim_gold.ipynb             # Build gold dimension tables
├── 3_medallion_processing_fact/
│   ├── 1_fact_bronze.ipynb          # Ingest order_items CSVs → bronze
│   ├── 2_fact_silver.ipynb          # Clean & transform fact data → silver
│   └── 3_fact_gold.ipynb            # Build gold fact table with derived metrics
└── 4_denormalization.dbquery.ipynb  # Final flat view for BI consumption
```

---

## Data Sources

| Entity        | Source Path                                      |
|---------------|--------------------------------------------------|
| Brands        | `/Volumes/ecommerce/source/raw/brands/*.csv`     |
| Categories    | `/Volumes/ecommerce/source/raw/category/*.csv`   |
| Products      | `/Volumes/ecommerce/source/raw/products/*.csv`   |
| Customers     | `/Volumes/ecommerce/source/raw/customers/*.csv`  |
| Date          | `/Volumes/ecommerce/source/raw/date/*.csv`       |
| Order Items   | `/Volumes/ecommerce/source/raw/order_items/landing/*.csv` |

---

## Layer Breakdown

### 🟫 Bronze — Raw Ingestion
- Schema-on-read using explicit `StructType` definitions
- All columns ingested as-is (strings where anomalies exist)
- Metadata columns added: `file_name`, `ingest_timestamp`
- Written as Delta tables with `mergeSchema=true`

### 🥈 Silver — Cleaning & Standardisation

| Table           | Key Transformations |
|-----------------|---------------------|
| `slv_brands`    | Trim whitespace, strip special chars from `brand_code` |
| `slv_category`  | Uppercase + trim `category_code`, drop duplicates |
| `slv_products`  | Strip `"g"` from weight, fix comma decimals in length, fix material spelling errors, abs() negative ratings |
| `slv_customers` | Drop null `customer_id` rows, fill null `phone` with `"NA"` |
| `slv_calendar`  | Parse date string → DateType, abs() negative week numbers, normalize `day_name` casing, enrich `quarter` and `week` labels |
| `slv_order_items` | Deduplicate on `(order_id, item_seq)`, cast types, fix `"Two"→2`, strip `$`/`%`, normalize channel & coupon |

### 🥇 Gold — Business-Ready Tables

| Table                   | Description |
|-------------------------|-------------|
| `gld_dim_products`      | Products joined with brand and category names |
| `gld_dim_customers`     | Customers enriched with region mapping (by country + state) |
| `gld_dim_calendar`      | Calendar with `date_id`, `month_name`, `is_weekend` flag |
| `gld_fact_order_items`  | Order items with `gross_amount`, `discount_amount`, `net_amount`, `net_amount_inr` (FX converted), `coupon_flag`, `date_id` |

#### FX Rates Used (fixed as of 2025-10-15)

| Currency | Rate to INR |
|----------|-------------|
| USD      | 88.29       |
| GBP      | 117.98      |
| AUD      | 57.55       |
| CAD      | 62.93       |
| AED      | 24.18       |
| SGD      | 68.18       |
| INR      | 1.00        |

---

## Denormalized View

`ecommerce.gold.fact_transactions_denorm` — A flat view joining fact + calendar + products, ready for direct BI tool connection.

**Includes:** transaction details, date attributes (year, month, quarter, week, is_weekend, hour_of_day), product attributes (SKU, category, brand, color, size, rating).

---

## Dashboard

<img width="1375" height="723" alt="image" src="https://github.com/user-attachments/assets/9ce353fc-c81b-498c-b113-fdaa5c82dc8b" />

---

## Tech Stack

- **Platform:** Databricks (Unity Catalog)
- **Storage Format:** Delta Lake
- **Language:** PySpark + Spark SQL
- **Catalog:** `ecommerce` → schemas: `bronze`, `silver`, `gold`

---
