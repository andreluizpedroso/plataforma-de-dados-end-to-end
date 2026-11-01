# Modern Data Engineering Platform

![CI](https://github.com/andreluizpedroso/plataforma-de-dados-end-to-end/actions/workflows/ci.yml/badge.svg)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![dbt](https://img.shields.io/badge/dbt-FF694B?style=flat&logo=dbt&logoColor=white)
![Airflow](https://img.shields.io/badge/Airflow-017CEE?style=flat&logo=apache-airflow&logoColor=white)
![DuckDB](https://img.shields.io/badge/DuckDB-FFF000?style=flat&logo=duckdb&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Metabase](https://img.shields.io/badge/Metabase-509EE3?style=flat&logo=metabase&logoColor=white)

🇧🇷 [Leia em português](README.pt-BR.md)

A hands-on reference implementation of an end-to-end data engineering platform that runs entirely on a local machine.

## Goal

Build, understand and evolve a pipeline with:

- PostgreSQL as the transactional OLTP database
- Python for ingestion and automation
- Parquet as columnar intermediate storage
- DuckDB as the local analytical warehouse
- dbt for SQL transformations
- Airflow for orchestration
- Metabase for BI
- Medallion architecture: bronze, silver and gold

## Why this project?

Learning data engineering from isolated tutorials leaves gaps: you know how to use each tool, but not how they fit together in a real pipeline. This project assembles the full environment (ingestion, transformation, orchestration and BI) on your own machine, with no cloud cost, so you can break, fix and understand each layer using data you control. It is the difference between studying one piece of the puzzle and seeing the whole picture assembled.

## Initial architecture

```text
PostgreSQL OLTP
      |
      | Python ingestion
      v
data/bronze/*.parquet
      |
      | cleaning, typing, deduplication
      v
data/silver/*.parquet
      |
      | dbt + DuckDB
      v
warehouse/local.duckdb
      |
      | gold marts
      v
Metabase
```

## Structure

```text
.
├── airflow/
│   ├── dags/
│   ├── logs/
│   └── plugins/
├── dashboards/
├── data/
│   ├── bronze/
│   ├── gold/
│   └── silver/
├── dbt/
├── docker/
│   └── postgres/
│       └── init/
├── docs/
├── ingestion/
│   └── src/
├── logs/
├── scripts/
├── tests/
├── warehouse/
├── .env.example
├── .gitignore
└── docker-compose.yml
```

## Getting started with the infrastructure

Copy `.env.example` to `.env` and adjust it if needed.

```powershell
docker compose up -d postgres_oltp metabase airflow-init airflow-webserver airflow-scheduler
```

Main services:

- PostgreSQL OLTP: `localhost:5432`
- Metabase: `http://localhost:3000`
- Airflow: `http://localhost:8080`

## Learning path

1. OLTP modeling in PostgreSQL.
2. Batch ingestion with Python.
3. Idempotent writes to bronze Parquet.
4. Cleaning and incremental processing in silver.
5. Gold dimensional modeling with dbt.
6. Orchestration with Airflow.
7. BI with Metabase.
8. Testing, observability and CI/CD.

## Initial domain

The first domain of the project is an investment platform.

Initial OLTP model:

- `app.customers`
- `app.accounts`
- `app.assets`
- `app.orders`
- `app.trades`
- `app.cash_transactions`

Documentation: `docs/oltp_investments_model.md`.

## Local batch pipeline

Once PostgreSQL is up and populated by the init scripts:

```powershell
python -m pip install -r requirements.txt
python -m ingestion.src.pipeline --step all
dbt run --project-dir dbt --profiles-dir dbt
dbt test --project-dir dbt --profiles-dir dbt
```

Airflow provides the `investments_batch_pipeline` DAG, which runs:

1. PostgreSQL ingestion -> Bronze/Silver/DuckDB
2. `dbt run`
3. `dbt test`
4. publishes `main_gold.mart_account_positions` to `analytics.mart_account_positions` in PostgreSQL for Metabase

Operational runbook: `docs/runbook.md`.

## Tests and CI

```bash
pytest -q
```

15 tests covering `ingestion/src/` (bronze, silver, warehouse, pipeline, publish, config, logging), with 99% coverage. Most of them run against real Parquet files and a real local DuckDB (no need for Postgres/Airflow to be running); the few parts that require Postgres (`bronze.extract_bronze`, `publish.publish_gold_to_postgres`) are tested with `psycopg2` mocks.

CI (GitHub Actions) runs `pytest` with coverage (60% minimum) and linting (`ruff`) on every push/PR to `main`.

## Next steps

- **Data quality with Great Expectations**: add data contracts between the bronze → silver → gold layers, with alerts when expectations are violated.
- **`dbt test` in CI**: CI currently runs the Python tests only; `dbt run` + `dbt test` still need to be added to the workflow (this requires starting a test Postgres on the runner).
- **Cloud deployment**: migrate the local warehouse to BigQuery or Redshift and the Parquet storage to S3/GCS, keeping the same Medallion architecture with controlled cost.
- **Streaming with Kafka**: introduce a producer that publishes trade events in real time and a consumer that feeds the bronze layer, evolving the pipeline from batch to near-real-time.
