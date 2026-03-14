# End-to-End Data Engineering Pipeline
### Airflow · dbt · BigQuery · Soda

A containerized ELT pipeline that ingests retail data into BigQuery, transforms it into analytics-ready models using dbt, and validates data quality with Soda — all orchestrated through Apache Airflow.

---

## Project Overview
This project demonstrates a full modern data stack:

| Tool | Role |
|------|------|
| **Airflow** | Orchestrates the pipeline with task dependencies and retry logic |
| **dbt** | Transforms raw landing zone data into dimension and fact tables |
| **Soda** | Runs automated data quality checks to catch schema issues and null propagation |
| **BigQuery** | Serves as the cloud data warehouse |
| **Docker** | Containerizes the entire workflow for reproducibility |

---

## Project Structure
```
├── dags/                  # Airflow DAGs (retail pipeline)
├── include/               # Supporting files and dbt models
├── Dockerfile             # Astro Runtime image
├── requirements.txt       # Python dependencies
└── airflow_settings.yaml  # Local Airflow connections and variables
```

---

## Running Locally

1. Install the [Astronomer CLI](https://docs.astronomer.io/astro/cli/install-cli)
2. Start the project:
```bash
astro dev start
```
3. Access the Airflow UI at `http://localhost:8080` (user: `admin`, password: `admin`)
4. Trigger the `retail` DAG and monitor task execution

---

## Requirements
- Docker Desktop
- Astronomer CLI
- Google Cloud account with BigQuery enabled
