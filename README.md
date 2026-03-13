Here's a clean, original version:

End-to-End Data Engineering Pipeline (Airflow, dbt, BigQuery, Soda)
A containerized ELT pipeline that ingests retail data into BigQuery, transforms it into analytics-ready models using dbt, and validates data quality with Soda — all orchestrated through Apache Airflow.
Project Overview
This project demonstrates a full modern data stack:

Airflow — orchestrates the pipeline with task dependencies and retry logic
dbt — transforms raw landing zone data into dimension and fact tables
Soda — runs automated data quality checks to catch schema issues and null propagation
BigQuery — serves as the cloud data warehouse
Docker — containerizes the entire workflow for reproducibility

Project Structure
├── dags/                  # Airflow DAGs (retail pipeline)
├── include/               # Supporting files and dbt models
├── Dockerfile             # Astro Runtime image
├── requirements.txt       # Python dependencies
├── packages.txt           # OS-level dependencies
└── airflow_settings.yaml  # Local Airflow connections and variables
Running Locally

Install the Astronomer CLI
Start the project:

bash   astro dev start

Access the Airflow UI at http://localhost:8080 (user: admin, password: admin)
Trigger the retail DAG and monitor task execution

Requirements

Docker Desktop
Astronomer CLI
Google Cloud account with BigQuery enabled
