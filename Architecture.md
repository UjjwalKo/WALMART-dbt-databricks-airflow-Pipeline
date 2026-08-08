# Architecture Documentation

## 1. Architectural Style

This project follows a layered data pipeline architecture with orchestration, transformation, and serving concerns separated into distinct components.

The design is inspired by a medallion architecture:

- Bronze/Source: Raw data available in Databricks
- Silver: Cleaned and standardized curated data
- Gold: Business-ready analytical datasets

---

## 2. High-Level Architecture

```text
Databricks Job (CDC Ingestion)
            │
            ▼
Airflow DAG: orchestrate
            │
            ├── Trigger Databricks job
            ├── Run dbt source freshness
            ├── Run dbt models (silver_t)
            ├── Run dbt tests (silver_t)
            ├── Run dbt models (silver_b)
            ├── Run dbt tests (silver_b)
            ├── Run gold ephemeral models
            ├── Run snapshots
            └── Run gold fact models
```

---

## 3. Component Responsibilities

### 3.1 Airflow Orchestrator

Airflow acts as the workflow controller. It coordinates the execution order of data tasks and provides visibility into job progress.

Responsibilities:

- Triggering ingestion jobs in Databricks
- Managing dependencies among transformation steps
- Running dbt commands in the correct sequence
- Handling failures through explicit task execution flow

### 3.2 dbt Transformation Engine

dbt is the transformation layer. It reads source tables, applies SQL-based transformations, and materializes models into the target warehouse schema.

Responsibilities:

- Building incremental models
- Joining and shaping data into business-ready structures
- Running tests on transformed data
- Ensuring maintainable SQL transformations through version-controlled code

### 3.3 Databricks Warehouse

Databricks acts as the execution and data storage environment. The project uses Databricks SQL Warehouse with a configured catalog and schema.

Responsibilities:

- Hosting the raw source tables
- Serving as the data warehouse target for dbt models
- Providing compute capacity for transformations

---

## 4. Data Flow

### Step 1: Ingestion Trigger

The workflow begins with the Airflow task ingest_cdc. This task connects to Databricks using the Databricks SDK and triggers a preconfigured Databricks job.

### Step 2: Source Freshness

After ingestion completes, dbt source freshness is run to validate that source tables are available and up to date.

### Step 3: Technical Silver Build

The technical silver models read from the source tables and create incremental representations with metadata timestamps.

### Step 4: Business Silver Build

The OBT model joins a set of technical silver tables into a consolidated dataset representing business-level entities and relationships.

### Step 5: Gold Layer Build

The gold layer consumes the OBT and creates curated data structures for reporting and analytics.

---

## 5. Model Layering Strategy

### Source Layer

Source models are declared in [walmart_project/models/source/sources.yml](walmart_project/models/source/sources.yml).

These represent external dependencies and keep the transformation logic decoupled from raw storage concerns.

### Silver Technical Models

These are entity-specific transformations:

- orders_t
- customer_t
- products_t
- employees_t
- order_item_t
- store_t

They are incremental and add a processed_at column.

### Silver Business Model

The OBT model unifies the technical silver entities into one combined structure for downstream processing.

### Gold Models

Gold models are grouped into:

- ephemeral: lightweight intermediate datasets
- fact: facts for analytics consumption

---

## 6. Incremental Design

The project uses incremental logic in the silver layer.

This allows the system to:

- Avoid recomputing the whole dataset each run
- Process only changed records based on updated_timestamp
- Improve runtime efficiency

The pattern is implemented using dbt’s is_incremental() macro and a conditional WHERE clause.

---

## 7. Dependency Graph

The transformation dependency chain can be summarized as:

```text
sources -> silver_t models -> obt_b -> gold ephemeral models -> gold fact model
```

This ensures that downstream models build only after their upstream dependencies are available.

---

## 8. Runtime Execution Pattern

### Airflow Execution Sequence

The DAG runs in a deterministic sequence:

1. Ingest data from Databricks
2. Clean old build artifacts
3. Check source freshness
4. Build technical silver models
5. Test technical silver models
6. Build business silver model
7. Test business silver model
8. Build gold models
9. Run snapshots
10. Build fact tables

This approach makes the workflow predictable and easy to observe from the Airflow UI.

---

## 9. Configuration Boundaries

### Airflow Configuration

Airflow is configured in:

- [airflow/docker-compose.yaml](airflow/docker-compose.yaml)
- [airflow/Dockerfile](airflow/Dockerfile)
- [airflow/dags/orchestrate.py](airflow/dags/orchestrate.py)

### dbt Configuration

dbt is configured in:

- [walmart_project/dbt_project.yml](walmart_project/dbt_project.yml)
- [walmart_project/profiles.yml](walmart_project/profiles.yml)

---

## 10. Security and Operational Considerations

The current implementation relies on environment variables for Databricks authentication and Airflow secrets. In a production environment, the following improvements are recommended:

- Use a secret manager such as AWS Secrets Manager, Azure Key Vault, or HashiCorp Vault
- Avoid hardcoding credentials in configuration files
- Define separate credentials for development, test, and production environments
- Lock down access to the Databricks SQL Warehouse and job endpoints

---

## 11. Observability and Maintenance

The project currently provides logs through:

- Airflow task logs
- dbt run and test logs
- Databricks job run history

Recommended enhancements:

- Add alerting on failed DAG executions
- Store run metadata in a central monitoring system
- Capture model freshness and data quality metrics

---

## 12. Scalability Outlook

The architecture is suitable for growth because it separates:

- orchestration logic from transformation logic
- reusable dbt models from execution control
- warehouse compute from application tooling

As the project grows, the pipeline can be expanded with:

- additional source systems
- more gold entities and marts
- automated CI/CD validation
- production-grade deployment practices

---

## 13. Summary

The architecture combines Airflow for orchestration, dbt for transformation, and Databricks for warehouse execution. The pipeline is structured around a layered model design that promotes maintainability, incremental processing, and analytical data readiness.
