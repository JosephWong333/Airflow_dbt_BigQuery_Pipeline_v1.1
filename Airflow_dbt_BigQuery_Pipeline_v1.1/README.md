# 🛒 End-to-End Retail Data Pipeline

An end-to-end data engineering pipeline that ingests raw online retail transaction data, models it into a star schema in Google BigQuery using dbt, enforces data quality at every stage with Soda Core, and surfaces business insights through a self-hosted Metabase dashboard — all orchestrated by Apache Airflow.

---

## 🏗️ Architecture & Stack

| Layer | Tool |
|---|---|
| Orchestration | Apache Airflow via Astro Runtime 9.2.0 |
| Ingestion | Astro Python SDK (`aql.load_file`) + GCS |
| Data Warehouse | Google BigQuery |
| Transformation | dbt-bigquery 1.5.3 (via `astronomer-cosmos`) |
| Data Quality | Soda Core for BigQuery 3.0.45 |
| Visualization | Metabase v0.46.6.4 |

**Dependency isolation:** To avoid version conflicts between dbt and Soda Core, the `Dockerfile` builds two separate Python virtual environments — `dbt_venv` and `soda_venv` — each activated only when their respective tasks run.

---

## 🔄 Pipeline Overview

The single `retail` DAG executes the following steps in order:

```
Upload CSV → GCS
    ↓
Create BigQuery Dataset
    ↓
Load raw_invoices (GCS → BigQuery)
    ↓
Soda Check: Raw Layer
    ↓
dbt Transform (Star Schema)
    ↓
Soda Check: Transform Layer
    ↓
dbt Report (Aggregations)
    ↓
Soda Check: Report Layer
```

### Source Data
The source is the [UCI Online Retail dataset](https://archive.ics.uci.edu/dataset/352/online+retail) — a 42MB CSV of transactional records from a UK-based gift and home décor retailer, covering international wholesale orders (primarily UK, with customers across Europe and beyond). Each row is an invoice line item with fields: `InvoiceNo`, `StockCode`, `Description`, `Quantity`, `InvoiceDate`, `UnitPrice`, `CustomerID`, and `Country`.

The CSV is uploaded from local storage to GCS (`online_retail_tham/raw/online_retail.csv`) and loaded into BigQuery as the `raw_invoices` table via the Astro Python SDK.

---

## 📐 Data Model

The dbt transform layer builds a **star schema** in the `retail` BigQuery dataset. All models materialize as tables.

### Dimension Tables
- **`dim_customer`** — Unique customers identified by a surrogate key on `CustomerID` + `Country`, enriched with ISO country codes via a `country` lookup table.
- **`dim_product`** — Unique products identified by a surrogate key on `StockCode` + `Description` + `UnitPrice` (since the same stock code can carry different prices).
- **`dim_datetime`** — Datetime dimension parsed from raw invoice date strings, extracting year, month, day, hour, minute, and weekday.

### Fact Table
- **`fct_invoices`** — Invoice line items joined to all three dimension tables, with `quantity` and `total` (quantity × unit price) as measures.

### Report Models
Three aggregation models sit on top of the fact table:
- **`report_customer_invoices`** — Top 10 countries by total revenue and invoice count.
- **`report_product_invoices`** — Top 10 products by total quantity sold.
- **`report_year_invoices`** — Monthly invoice count and revenue trends.

---

## 🛡️ Data Quality

Soda Core runs automated checks at **three checkpoints** in the pipeline, each triggered as an `@task.external_python` call inside the `soda_venv`:

| Checkpoint | Scope | Example Checks |
|---|---|---|
| `check_load` | `raw_invoices` | Required columns present, correct data types |
| `check_transform` | `dim_*`, `fct_invoices` | No duplicate/null surrogate keys, valid weekday range (0–6), no negative invoice totals |
| `check_report` | `report_*` | No null countries or stock codes, no zero/negative revenue or quantity totals |

If any check fails, the Soda scan raises a `ValueError` and the DAG task fails immediately, blocking downstream steps.

---

## 📂 Project Structure

```
├── dags/
│   └── retail.py                   # Main Airflow DAG
├── include/
│   ├── data/
│   │   └── online_retail.csv       # Source dataset (not committed)
│   ├── dbt/
│   │   ├── models/
│   │   │   ├── transform/          # dim_customer, dim_datetime, dim_product, fct_invoices
│   │   │   └── report/             # report_customer_invoices, report_product_invoices, report_year_invoices
│   │   ├── cosmos_config.py        # Cosmos ProfileConfig + ProjectConfig
│   │   ├── dbt_project.yml
│   │   ├── packages.yml            # dbt-utils 1.1.1
│   │   └── profiles.yml            # BigQuery service-account auth
│   ├── gcp/
│   │   └── service_account.json    # GCP credentials (not committed)
│   ├── metabase-data/              # Persistent Metabase volume (not committed)
│   └── soda/
│       ├── check_function.py       # Reusable Soda scan runner
│       ├── configuration.yml       # Soda BigQuery connection (credentials not committed)
│       └── checks/
│           ├── sources/            # raw_invoices.yml
│           ├── transform/          # dim_*.yml, fct_invoices.yml
│           └── report/             # report_*.yml
├── tests/
│   ├── test_dag_integrity_default.py   # Astro dev parse test (do not edit)
│   └── test_dag_integrity.py           # Custom tests: tags, retries, import errors
├── Dockerfile                      # Astro Runtime + dbt_venv + soda_venv
├── docker-compose.override.yml     # Metabase service on port 3000
├── requirements.txt                # astronomer-cosmos[dbt-bigquery], protobuf
└── packages.txt                    # OS-level packages (none currently required)
```

---

## 🚀 Getting Started

### Prerequisites
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Astro CLI](https://docs.astronomer.io/astro/cli/install-cli)
- A GCP project with BigQuery enabled and a service account with BigQuery Admin permissions

### Setup

1. **Place your GCP service account key** at `include/gcp/service_account.json`.

2. **Configure the Airflow GCP connection.** In the Airflow UI under **Admin → Connections**, create a connection with:
   - Connection ID: `gcp`
   - Connection Type: `Google Cloud`
   - Keyfile Path: `/usr/local/airflow/include/gcp/service_account.json`

3. **Add your Soda Cloud credentials** to `include/soda/configuration.yml` (do not commit these — use environment variables or Airflow Variables in production).

4. **Place the source CSV** at `include/data/online_retail.csv`.

5. **Start the stack:**
   ```bash
   astro dev start
   ```
   This launches the Airflow Scheduler, Webserver, Triggerer, Postgres metadata DB, and Metabase.

### Access

| Interface | URL | Credentials |
|---|---|---|
| Airflow UI | http://localhost:8080 | `admin` / `admin` |
| Metabase | http://localhost:3000 | Set on first launch |
| Postgres | `localhost:5432/postgres` | — |

---

