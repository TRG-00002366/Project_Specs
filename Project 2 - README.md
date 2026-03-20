# Project 2: End-to-End E-Commerce Data Pipeline

> **Kafka / S3 → PySpark → Snowflake → Power BI / Streamlit → Airflow Orchestration**

An end-to-end data pipeline that ingests streams via **Apache Kafka** or **CSV files from AWS S3**, processes and transforms them using **PySpark DataFrames / SparkSQL**, loads into a **Snowflake Data Warehouse** for **Power BI** visualization, and orchestrates the entire workflow via **Apache Airflow**.

---

## Architecture

```
  ┌─────────────┐     ┌───────────┐     ┌──────────────────┐     ┌───────────┐
  │ Faker/CSV   │────▶│ Kafka     │────▶│ Spark Streaming  │────▶│ AWS S3    │
  │ Producer    │     │ Topic     │     │ Consumer         │     │ (Parquet) │
  └──────┬──────┘     └───────────┘     └──────────────────┘     └─────┬─────┘
         │                                                              │
         └──────── CSV upload ──────────────────────────────────────────┘
                                                                        │
                                                               ┌────────▼────────┐
                                                               │ PySpark Batch   │
                                                               │ ETL (SparkSQL)  │
                                                               └────────┬────────┘
                                                                        │
                                                                        ▼
  ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────────────┐
  │ Power BI         │◀────│ Snowflake Gold   │◀────│ Snowflake Bronze         │
  │ Dashboards       │     │ (Star Schema)    │     │ (S3 External Stage)      │
  └──────────────────┘     │ via dbt          │     │ COPY INTO / Snowpipe     │
  ┌──────────────────┐     └──────────────────┘     └──────────────────────────┘
  │ Streamlit App    │◀────────────┘
  └──────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────┐
  │         Apache Airflow — Unified DAG (single orchestrator)              │
  │  start → produce → stream → batch_etl → snowflake_load → dbt → end    │
  └──────────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Docker & Docker Compose | [Install Docker](https://docs.docker.com/get-docker/) |
| AWS Account | S3 bucket + IAM credentials (Access Key + Secret) |
| Snowflake Account | [30-day free trial](https://signup.snowflake.com/) |
| SnowSQL CLI | [Install](https://docs.snowflake.com/en/user-guide/snowsql-install-config) |
| Python 3.10+ | For dbt and Streamlit |
| Power BI Desktop | Windows — [Download](https://powerbi.microsoft.com/desktop/) |

---

## Project Structure

```
project2/
├── docker-compose.yml              # Kafka, Airflow, Postgres
├── Dockerfile                      # Custom Airflow + Spark image
├── requirements.txt                # Python dependencies
├── .env.example                    # AWS + Snowflake credentials template
├── config/
│   └── pipeline_config.py          # Centralized configuration
├── kafka/
│   └── producer.py                 # Multi-mode: Kafka + S3 + local
├── spark/
│   ├── s3_ingest.py                # Read CSV from S3 → Parquet
│   ├── stream_consumer.py          # Kafka → S3/local Parquet
│   ├── batch_etl.py                # SparkSQL transforms (4 datasets)
│   └── snowflake_loader.py         # S3 external stage → Snowflake COPY INTO
├── snowflake/                      # Run in order (01–08)
│   ├── 01_setup_warehouse.sql
│   ├── 02_create_bronze_tables.sql
│   ├── 03_s3_external_stage.sql    # S3 stages, Snowpipe, VARIANT, clone
│   ├── 04_udfs.sql
│   ├── 05_streams_and_tasks.sql    # CDC + Silver auto-processing
│   ├── 06_gold_star_schema.sql     # Fact + 4 dimensions + marts
│   ├── 07_rbac.sql
│   └── 08_masking_policies.sql
├── dbt_ecommerce/
│   ├── dbt_project.yml / profiles.yml / packages.yml
│   ├── models/staging/             # stg_orders, stg_regions
│   ├── models/marts/               # dim_*, fact_orders, mart_*
│   ├── models/schema.yml           # Tests (unique, not_null, relationships)
│   ├── seeds/regions.csv
│   ├── snapshots/snap_orders.sql   # SCD Type 2
│   └── tests/                      # 3 custom singular tests
├── dashboards/
│   ├── powerbi_guide.md            # Full Power BI implementation guide
│   ├── streamlit_app.py            # Live monitoring dashboard
│   ├── requirements.txt
│   └── .streamlit/secrets.toml
├── airflow/dags/
│   └── ecommerce_unified_dag.py    # SINGLE unified end-to-end DAG
├── ci_cd/.github/workflows/
│   └── dbt_ci.yml
├── GOVERNANCE.md
└── README.md
```

---

## Step-by-Step Setup

### Step 1: Configure Credentials

```bash
cd projects/project2
cp .env.example .env
```

Edit `.env` with your AWS and Snowflake credentials:
```bash
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
S3_BUCKET=ecommerce-data-pipeline
SNOWFLAKE_ACCOUNT=abc12345.us-east-1
SNOWFLAKE_USER=your_username
SNOWFLAKE_PASSWORD=your_password
```

### Step 2: Create S3 Bucket

```bash
aws s3 mb s3://ecommerce-data-pipeline --region us-east-1
```

### Step 3: Set Up Snowflake

Sign up at https://signup.snowflake.com/ → Run scripts in order via Snowflake Worksheet or SnowSQL:

```bash
snowsql -f snowflake/01_setup_warehouse.sql
snowsql -f snowflake/02_create_bronze_tables.sql
```

Edit `snowflake/03_s3_external_stage.sql` — replace `<your-bucket>`, `<your-key>`, `<your-secret>` with actual values, then run:

```bash
snowsql -f snowflake/03_s3_external_stage.sql
snowsql -f snowflake/04_udfs.sql
snowsql -f snowflake/05_streams_and_tasks.sql
snowsql -f snowflake/06_gold_star_schema.sql
```

### Step 4: Start Docker Environment

```bash
docker compose build
docker compose up -d
```

Wait ~60 seconds for services to start, then verify:
```bash
docker compose ps           # All services should be "Up"
```

Airflow UI: http://localhost:8080 (admin / admin)

---

## Step-by-Step Execution

### Phase 1: Produce Data (Kafka + S3)

```bash
# Generate 500 events → Kafka topic + S3 CSV
docker compose exec airflow-scheduler \
    python /opt/airflow/kafka/producer.py --mode both --num-events 500
```

Verify S3:
```bash
aws s3 ls s3://ecommerce-data-pipeline/raw/csv/
```

### Phase 2: Spark Streaming (Kafka → S3)

```bash
docker compose exec airflow-scheduler \
    spark-submit --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0,org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 \
    /opt/airflow/spark/stream_consumer.py --output s3 --duration 120
```

### Phase 3: Batch ETL (PySpark DataFrame + SparkSQL)

```bash
docker compose exec airflow-scheduler \
    spark-submit --packages org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 \
    /opt/airflow/spark/batch_etl.py --source s3
```

Produces: `hourly_sales_summary`, `top_10_products`, `regional_revenue`, `order_status_breakdown`

### Phase 4: Load into Snowflake

```bash
docker compose exec airflow-scheduler \
    python /opt/airflow/spark/snowflake_loader.py --source s3 --load-transformed
```

Verify:
```sql
SELECT COUNT(*) FROM BRONZE.RAW_ORDERS;
SELECT COUNT(*) FROM GOLD.FACT_ORDERS;
```

### Phase 5: dbt Transforms

```bash
cd dbt_ecommerce
pip install dbt-snowflake==1.7.*
dbt deps
dbt seed
dbt snapshot
dbt run
dbt test
dbt docs generate && dbt docs serve    # View lineage at http://localhost:8080
```

### Phase 6: Power BI Dashboard

Follow the complete guide at **`dashboards/powerbi_guide.md`** which covers:

1. **Connect** Power BI to Snowflake Gold layer
2. **Schema** — Set up star schema relationships in Model View
3. **DAX Measures** — 18 measures (Revenue, YTD, MoM Growth, Cancel Rate, etc.)
4. **Reports** — 4 pages: Executive Summary, Product Analytics, Regional, Data Alerts
5. **Conditional Formatting** — Red/Yellow/Green on cancellation rates
6. **Data Alerts** — Automated alerts on KPI thresholds
7. **Scheduled Refresh** — Daily auto-refresh from Snowflake
8. **DAX Studio** — Performance monitoring and query optimization

### Phase 7: Streamlit Dashboard

```bash
cd dashboards
pip install -r requirements.txt
# Edit .streamlit/secrets.toml with your Snowflake credentials
streamlit run streamlit_app.py
```

Open http://localhost:8501 — features: KPIs, charts, filters, outlier detection

### Phase 8: Governance & Security

```bash
snowsql -f snowflake/07_rbac.sql
snowsql -f snowflake/08_masking_policies.sql
```

Test masking:
```sql
USE ROLE DATA_ANALYST;
SELECT customer_email FROM SILVER.CLEAN_ORDERS LIMIT 5;  -- shows ***@domain.com

USE ROLE DATA_STEWARD;
SELECT customer_email FROM SILVER.CLEAN_ORDERS LIMIT 5;  -- shows real emails
```

Review `GOVERNANCE.md` for data classification, RBAC matrix, GDPR compliance.

---

## Running via Airflow (Unified DAG)

Instead of running each phase manually, use the **unified DAG**:

1. Open Airflow UI: http://localhost:8080
2. Find DAG: `ecommerce_unified_pipeline`
3. Click **Trigger DAG** with parameters:
   - `ingest_mode`: `both` (Kafka + S3), `kafka`, or `s3`
   - `source`: `s3` (recommended) or `local`
4. The DAG runs the **entire pipeline** automatically:

```
start → produce_to_kafka ──→ stream_consumer ──┐
      → ingest_from_s3 ───────────────────────┤
                                               ▼
                                         batch_etl
                                               │
                                         load_to_snowflake
                                               │
                                 dbt_deps → seed → snapshot → run → test
                                               │
                                       ┌───────┴───────┐
                                    dbt_docs      validate → end
```

---

## Curriculum Mapping

| Pipeline Step | Curriculum Topic | Week |
|---------------|-----------------|:----:|
| Kafka Producer | Apache Kafka | 3 |
| Spark Streaming | Structured Streaming | 3 |
| PySpark Batch ETL | DataFrames, SparkSQL, Window Functions | 1-2 |
| S3 Integration | AWS, hadoop-aws connector | 1 |
| Snowflake Setup | Warehouse, Schemas, SnowSQL | 5 |
| Snowpipe, Stages | Bulk loading, File Formats | 5 |
| VARIANT Queries | Semi-structured data | 5 |
| Streams & Tasks | CDC, Automation | 5 |
| Star Schema | Dimensions, Facts | 5 |
| dbt Models | Staging, Marts, ref(), Seeds, Snapshots | 5 |
| dbt Tests | Data Quality | 7 |
| Power BI | DAX, Reports, Alerts, Refresh | 6 |
| Streamlit | Connect to Snowflake, Charts, Outliers | 6 |
| RBAC & Masking | Governance, PII | 7 |
| CI/CD | DataOps, Automated Testing | 7 |
| Airflow DAG | Unified Orchestration | 4 |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| S3 access denied | Check AWS credentials in `.env`, verify IAM permissions |
| Snowflake connection failed | Verify account URL format: `account.region` |
| Kafka not ready | Wait 30s after `docker compose up`, check `docker compose logs kafka` |
| Spark can't read S3 | Ensure `hadoop-aws` and `aws-java-sdk-bundle` packages in `--packages` |
| dbt "relation does not exist" | Run Snowflake scripts 01-06 and `snowflake_loader.py` first |
| Power BI can't connect | Use server: `<account>.snowflakecomputing.com`, check firewall |
| Streamlit error | Check `.streamlit/secrets.toml` credentials |
| Masking not working | Use `USE ROLE DATA_ANALYST` to see masked values |
