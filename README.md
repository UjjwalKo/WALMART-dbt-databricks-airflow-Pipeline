# Data Project DBT Documentation

## 1. Overview

This project implements a modern data engineering workflow centered around dbt, Airflow, and Databricks. It is designed to ingest data from a Databricks-backed bronze layer, transform it through silver-layer models, and expose curated gold-layer analytics models for downstream reporting and consumption.

The project is organized as a dbt project named Walmart Project, orchestrated by Apache Airflow with Docker-based deployment. The end-to-end pipeline performs:

- CDC-triggered ingestion through Databricks jobs
- Source freshness checks
- dbt model execution and testing
- Gold-layer model creation for analytical consumption

---

## 2. Project Goals

The main objectives of this project are to:

1. Create a structured, maintainable transformation pipeline using dbt.
2. Orchestrate the workflow reliably with Apache Airflow.
3. Use Databricks SQL Warehouse as the compute and storage target.
4. Build layered data models following a medallion-style architecture:
   - Source
   - Silver
   - Gold
5. Support incremental processing for large or frequently updated datasets.

---

## 3. Technology Stack

### Core Components

- Python 3.11+
- Apache Airflow 3.3.0
- dbt Core 1.12.0
- dbt-databricks 1.10.9
- Databricks SDK
- Databricks SQL Warehouse
- Docker and Docker Compose

### Main Libraries

The project dependencies are specified in [pyproject.toml](pyproject.toml) and include:

- airflow-operators
- apache-airflow
- databricks-sdk
- dbt-core
- dbt-databricks

---

## 4. Repository Structure

```text
.
├── airflow/
│   ├── dags/
│   │   └── orchestrate.py
│   ├── Dockerfile
│   ├── docker-compose.yaml
│   ├── requirements.txt
│   └── config/
├── walmart_project/
│   ├── dbt_project.yml
│   ├── profiles.yml
│   ├── macros/
│   │   └── custom_schema.sql
│   ├── models/
│   │   ├── source/
│   │   ├── silver_t/
│   │   ├── silver_b/
│   │   ├── gold/
│   │   └── gold/ephemeral
│   ├── snapshots/
│   ├── seeds/
│   ├── tests/
│   └── target/
├── README.md
├── pyproject.toml
└── logs/
```

### Key Directories

- airflow/: Contains Airflow deployment files and orchestration DAGs.
- walmart_project/: Contains the dbt project and all transformation logic.
- logs/: Stores execution and runtime logs.

---

## 5. Architecture Overview

The system follows a layered architectural pattern:

1. Source Layer
   - Raw data is assumed to exist in a Databricks bronze schema.
   - dbt sources are defined for tables such as orders, customers, products, order_items, stores, and employees.

2. Silver Layer
   - Technical silver models create incremental copies of source tables with added metadata such as processed_at.
   - Business silver model builds a consolidated object table (OBT) by joining the silver entities.

3. Gold Layer
   - Ephemeral gold models are created from the OBT.
   - Fact and dimension-like analytical tables are derived for reporting use cases.

4. Orchestration Layer
   - Airflow triggers an external Databricks job for CDC ingestion.
   - After ingestion, Airflow runs dbt commands to refresh the warehouse models.

---

## 6. Airflow Workflow

The Airflow DAG is defined in [airflow/dags/orchestrate.py](airflow/dags/orchestrate.py).

### DAG Behavior

The workflow executes the following steps in order:

1. Trigger a Databricks job for CDC ingestion.
2. Wait until the Databricks job completes successfully.
3. Clean the dbt target and logs directories.
4. Run dbt source freshness checks.
5. Execute silver technical models.
6. Run tests for the silver technical models.
7. Execute silver business models.
8. Run tests for the silver business models.
9. Build the gold ephemeral models.
10. Run snapshots.
11. Build gold fact models.

### Airflow Components

- DAG: orchestrate
- Tasks:
  - ingest_cdc
  - clean_target
  - source_freshness
  - silver_technical
  - silver_technical_test
  - silver_business
  - silver_business_test
  - gold_ephemeral
  - gold_dimensions
  - gold_fact

### Environment Dependencies

The DAG uses environment variables:

- DATABRICKS_HOST
- DATABRICKS_TOKEN
- DATABRICKS_JOB_ID

These values are required for the Databricks integration.

---

## 7. dbt Project Configuration

### Project Name

The project is named walmart_project and is configured in [walmart_project/dbt_project.yml](walmart_project/dbt_project.yml).

### Key Configuration Details

- Profile: walmart_project
- Model paths: models/
- Analysis paths: analyses/
- Test paths: tests/
- Seed paths: seeds/
- Macro paths: macros/
- Snapshot paths: snapshots/

### Model Materialization Rules

The dbt project configures:

- silver_t models as views in the silver_t schema
- silver_b models as views in the silver_b schema
- gold models as views in the gold schema
- ephemeral gold models as ephemeral materialization

---

## 8. Source Configuration

The source definitions are stored in [walmart_project/models/source/sources.yml](walmart_project/models/source/sources.yml).

### Declared Sources

The project references the following source tables:

- orders
- customers
- products
- order_items
- stores
- employees

### Source Connection

The dbt profile points to Databricks using:

- catalog: walmart
- schema: dbt_schema
- type: databricks

---

## 9. Data Modeling Layers

### 9.1 Source Layer

The source layer describes the raw business tables coming from Databricks. These are not transformed in place; they are referenced as sources by dbt models.

### 9.2 Silver Layer

The silver layer is divided into:

- Technical silver models: [walmart_project/models/silver_t](walmart_project/models/silver_t)
- Business silver model: [walmart_project/models/silver_b](walmart_project/models/silver_b)

#### Technical Silver Models

Each technical silver model:

- reads from a source table
- applies incremental logic
- adds a processed_at timestamp
- uses a natural key as the unique key

Examples include:

- orders_t.sql
- customer_t.sql
- products_t.sql
- employees_t.sql
- order_item_t.sql
- store_t.sql

#### Business Silver Model

The model [walmart_project/models/silver_b/obt_b.sql](walmart_project/models/silver_b/obt_b.sql) creates a wide business-oriented table by joining multiple technical silver models. This acts as a consolidated object table for downstream gold models.

### 9.3 Gold Layer

The gold layer is composed of:

- Ephemeral models in [walmart_project/models/gold/ephemeral](walmart_project/models/gold/ephemeral)
- Fact model in [walmart_project/models/gold/fact](walmart_project/models/gold/fact)

These models are intended to provide curated outputs for analytics and reporting.

---

## 10. Incremental Processing

Several silver models are configured as incremental. This means the pipeline does not always rebuild all data from scratch. Instead, it only processes new or updated records based on the updated_timestamp column.

### Incremental Pattern

The models use logic similar to:

- materialized = incremental
- unique_key = business key
- conditional filtering on updated_timestamp

This helps improve efficiency and reduce rebuild cost.

---

## 11. Custom Schema Macro

The custom schema macro in [walmart_project/macros/custom_schema.sql](walmart_project/macros/custom_schema.sql) enables dbt to map custom schema names to target schemas. This is useful for separating logical layers and ensuring clarity across development and production environments.

---

## 12. Testing Strategy

The project includes a basic dbt test in [walmart_project/tests/test_obt_b.sql](walmart_project/tests/test_obt_b.sql).

### Current Test Behavior

The test raises a warning when the consolidated OBT model contains null values in critical IDs such as:

- order_id
- product_id
- employee_id
- store_id
- order_item_id
- customer_id

This acts as a lightweight data quality safeguard.

---

## 13. Deployment and Environment Setup

### Local Development

The repository uses a Python environment managed with uv/virtualenv. The environment is defined in [pyproject.toml](pyproject.toml).

### Dockerized Airflow

The Airflow deployment uses Docker Compose configured in [airflow/docker-compose.yaml](airflow/docker-compose.yaml).

It includes:

- PostgreSQL for Airflow metadata
- Redis for Celery backend
- Airflow API server
- Scheduler
- Dag processor
- Worker
- Triggerer

### Running the Project

To work with this project locally, a user would typically:

1. Install dependencies.
2. Configure Databricks environment variables.
3. Start the Airflow stack with Docker Compose.
4. Trigger the orchestrate DAG.
5. Monitor through the Airflow UI and dbt logs.

---

## 14. Operational Notes

### Strengths

- Clear separation between orchestration and transformation logic
- Layered dbt architecture
- Incremental processing support
- Integration with Databricks and Airflow

### Areas to Improve

- Add stronger data quality tests
- Introduce more explicit documentation for each model
- Add environment-specific profiles for dev/test/prod
- Consider storing secrets securely rather than relying only on environment variables
- Add CI/CD automation for dbt validation and deployment

---

## 15. Summary

This project demonstrates a practical implementation of an ELT-style data pipeline using dbt and Airflow. It connects to Databricks, builds layered transformations, and orchestrates the workflow from ingestion to reporting-ready data. The current setup is well-suited for development and learning purposes, and it can be extended into a production-grade data platform with stronger governance and operational controls.
