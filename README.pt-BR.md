# Modern Data Engineering Platform

![CI](https://github.com/andreluizpedroso/plataforma-de-dados-end-to-end/actions/workflows/ci.yml/badge.svg)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![dbt](https://img.shields.io/badge/dbt-FF694B?style=flat&logo=dbt&logoColor=white)
![Airflow](https://img.shields.io/badge/Airflow-017CEE?style=flat&logo=apache-airflow&logoColor=white)
![DuckDB](https://img.shields.io/badge/DuckDB-FFF000?style=flat&logo=duckdb&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Metabase](https://img.shields.io/badge/Metabase-509EE3?style=flat&logo=metabase&logoColor=white)

🇺🇸 [Read in English](README.md)

Projeto educacional para construir uma plataforma local de engenharia de dados ponta a ponta.

## Objetivo

Construir, entender e evoluir um pipeline com:

- PostgreSQL como banco transacional OLTP
- Python para ingestao e automacao
- Parquet como armazenamento intermediario columnar
- DuckDB como warehouse analitico local
- dbt para transformacoes SQL
- Airflow para orquestracao
- Metabase para BI
- Arquitetura Medallion: bronze, silver e gold

## Por que esse projeto?

Aprender engenharia de dados lendo tutoriais isolados deixa lacunas: você sabe usar cada ferramenta, mas não sabe como elas se encaixam num pipeline real. Este projeto monta o ambiente completo — ingestão, transformação, orquestração e BI — na sua máquina, sem custo de nuvem, para que você possa quebrar, consertar e entender cada camada com dados que você mesmo controla. É a diferença entre estudar uma peça do quebra-cabeça e ver a imagem toda montada.

## Arquitetura inicial

```text
PostgreSQL OLTP
      |
      | Python ingestion
      v
data/bronze/*.parquet
      |
      | limpeza, tipagem, deduplicacao
      v
data/silver/*.parquet
      |
      | dbt + DuckDB
      v
warehouse/local.duckdb
      |
      | marts gold
      v
Metabase
```

## Estrutura

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

## Como iniciar a infraestrutura

Copie `.env.example` para `.env` e ajuste se quiser.

```powershell
docker compose up -d postgres_oltp metabase airflow-init airflow-webserver airflow-scheduler
```

Servicos principais:

- PostgreSQL OLTP: `localhost:5432`
- Metabase: `http://localhost:3000`
- Airflow: `http://localhost:8080`

## Ordem de aprendizado

1. Modelagem OLTP no PostgreSQL.
2. Ingestao batch com Python.
3. Escrita idempotente em Parquet bronze.
4. Limpeza e incrementalidade em silver.
5. Modelagem dimensional gold com dbt.
6. Orquestracao com Airflow.
7. BI com Metabase.
8. Testes, observabilidade e CI/CD.

## Dominio inicial

O primeiro dominio do projeto e uma plataforma de investimentos.

Modelo OLTP inicial:

- `app.customers`
- `app.accounts`
- `app.assets`
- `app.orders`
- `app.trades`
- `app.cash_transactions`

Documentacao: `docs/oltp_investments_model.md`.

## Pipeline batch local

Depois que o PostgreSQL estiver no ar e populado pelos scripts de init:

```powershell
python -m pip install -r requirements.txt
python -m ingestion.src.pipeline --step all
dbt run --project-dir dbt --profiles-dir dbt
dbt test --project-dir dbt --profiles-dir dbt
```

O Airflow possui a DAG `investments_batch_pipeline`, que executa:

1. ingestao PostgreSQL -> Bronze/Silver/DuckDB
2. `dbt run`
3. `dbt test`
4. publica `main_gold.mart_account_positions` em `analytics.mart_account_positions` no PostgreSQL para o Metabase

Runbook operacional: `docs/runbook.md`.

## Testes e CI

```bash
pytest -q
```

15 testes cobrindo `ingestion/src/` (bronze, silver, warehouse, pipeline, publish, config, logging) — 99% de cobertura. A maioria roda contra arquivos Parquet e um DuckDB local reais (sem precisar do Postgres/Airflow no ar); as poucas partes que exigem Postgres (`bronze.extract_bronze`, `publish.publish_gold_to_postgres`) são testadas com mocks do `psycopg2`.

CI (GitHub Actions) roda `pytest` com cobertura (mínimo 60%) e lint (`ruff`) em todo push/PR para `main`.

## Próximos passos

- **Qualidade de dados com Great Expectations**: adicionar contratos de dados entre as camadas bronze → silver → gold, com alertas quando expectativas forem violadas.
- **`dbt test` no CI**: hoje o CI roda os testes Python; falta incluir `dbt run` + `dbt test` no workflow (exige subir um Postgres de teste no runner).
- **Deploy em nuvem**: migrar o warehouse local para BigQuery ou Redshift e o storage de Parquet para S3/GCS, mantendo a mesma arquitetura Medallion com custo controlado.
- **Streaming com Kafka**: introduzir um producer que publique eventos de trade em tempo real e um consumer que alimente a camada bronze, evoluindo o pipeline de batch para near-realtime.
