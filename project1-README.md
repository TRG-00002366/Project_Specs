# Project 1 — Scenario A: Kafka Producer (Faker-Generated Data)

End-to-end e-commerce order analytics pipeline using **PySpark**, **Kafka**, and **Airflow**, all running on Docker.

---

## Architecture

```
Producer (Faker) → Kafka Topic → Spark Streaming → Raw Parquet
                                                        ↓
                                         ┌──────────────┴──────────────┐
                                         ↓                             ↓
                                   RDD Batch ETL              DataFrame Batch ETL
                                   (revenue by product)       (4 analytics datasets)
                                         └──────────────┬──────────────┘
                                                        ↓
                                                  Airflow DAG
                                              (orchestration)
```

---

## Prerequisites

- **Docker Desktop** installed and running
- **Docker Compose** v2+ (included with Docker Desktop)
- **8 GB+ RAM** allocated to Docker (Spark + Kafka + Airflow are memory-intensive)

---

## Project Structure

```
project1-kafka-producer/
├── docker-compose.yml          # All services: Kafka (KRaft), Airflow, Postgres
├── Dockerfile                  # Custom Airflow image with PySpark + Java
├── requirements.txt            # Python dependencies
├── README.md                   # This file
├── data/
│   └── regions.csv             # Reference dimension table
├── kafka/
│   └── producer.py             # Faker-based Kafka event producer
├── spark/
│   ├── stream_consumer.py      # Spark Structured Streaming consumer
│   ├── batch_rdd_etl.py        # RDD-based batch processing
│   └── batch_df_etl.py         # DataFrame/Spark SQL batch processing
└── airflow/
    └── dags/
        └── ecommerce_dag.py    # Airflow orchestration DAG
```

---

## Step-by-Step Execution Guide

### Step 1: Start the Environment

```bash
# Navigate to the project directory
cd project1-kafka-producer

# Build the custom image and start all services
docker compose build
docker compose up -d
```

Wait ~60 seconds for all services to initialize. Check status:

```bash
docker compose ps
```

All services should show `running` (except `airflow-init` which exits after setup).

### Step 2: Verify Services

| Service            | URL / Check                                 |
|--------------------|--------------------------------------------- |
| **Airflow UI**     | http://localhost:8080 (user: `admin`, pass: `admin`) |
| **Spark Master UI**| http://localhost:8180 |
| **Spark Worker UI**| http://localhost:8181 |
| **Kafka**          | `docker compose exec kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list` |
| **Postgres**       | Port 5432 (used by Airflow internally)       |

### Step 3: Run the Kafka Producer

```bash
# Produce 500 order events to the 'ecommerce_orders' topic
docker compose exec airflow-scheduler python /opt/airflow/kafka/producer.py --num-events 500
```

Verify events were produced:

```bash
docker compose exec kafka kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic ecommerce_orders \
    --from-beginning \
    --max-messages 5
```

### Step 4: Run the Spark Streaming Consumer

```bash
docker compose exec airflow-scheduler spark-submit \
    --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0 \
    /opt/airflow/spark/stream_consumer.py \
    --bootstrap-servers kafka:9092 \
    --duration 120
```

This will consume messages for 120 seconds and write raw Parquet files to `data/raw/orders/`.

### Step 5: Run the Batch RDD ETL

```bash
docker compose exec airflow-scheduler spark-submit \
    /opt/airflow/spark/batch_rdd_etl.py
```

Output: `data/transformed/rdd_revenue_by_product/` — revenue per product as text files.

### Step 6: Run the Batch DataFrame ETL

```bash
docker compose exec airflow-scheduler spark-submit \
    /opt/airflow/spark/batch_df_etl.py
```

Outputs (all Parquet):
- `data/transformed/hourly_sales_summary/`
- `data/transformed/top_10_products/`
- `data/transformed/regional_revenue/`
- `data/transformed/order_status_breakdown/`

### Step 7: Run via Airflow (Full Orchestration)

Instead of running each step manually, trigger the DAG from the Airflow UI:

1. Open http://localhost:8080
2. Log in with `admin` / `admin`
3. Find the `ecommerce_pipeline` DAG
4. Toggle it **ON** (if paused)
5. Click **Trigger DAG** (▶ button)
6. Monitor task execution in the **Graph** view

Or trigger from the CLI:

```bash
docker compose exec airflow-scheduler airflow dags trigger ecommerce_pipeline
```

### Step 8: Inspect Output Data

```bash
# Check that output directories have data
docker compose exec airflow-scheduler ls -la /opt/airflow/data/raw/orders/
docker compose exec airflow-scheduler ls -la /opt/airflow/data/transformed/
```

### Step 9: Tear Down

```bash
# Stop all services and remove volumes
docker compose down -v
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Kafka broker not ready | Wait 30–60 seconds after `docker compose up`. The producer has retry logic. |
| Spark Streaming hangs | Ensure Kafka topic has messages. Check `--duration` flag is set. |
| Airflow DAG not visible | Check for Python syntax errors: `docker compose exec airflow-scheduler python -c "import dags.ecommerce_dag"` |
| Out of memory | Increase Docker Desktop memory to 8 GB+. |
| Port conflicts | Change ports in `docker-compose.yml` if 8080, 9092, or 5432 are in use. |
| Kafka scripts not found | In the `apache/kafka` image, scripts are at `/opt/kafka/bin/`. Use the full path. |

---

## Key Concepts Demonstrated

| Week | Concepts | Where in Code |
|------|----------|---------------|
| **1** | RDDs, map, filter, reduceByKey, accumulators, spark-submit | `spark/batch_rdd_etl.py` |
| **2** | DataFrames, Spark SQL, joins, window functions, pivot, caching | `spark/batch_df_etl.py` |
| **3** | Kafka topics, producers, consumers, Spark Streaming | `kafka/producer.py`, `spark/stream_consumer.py` |
| **3** | Spark Standalone cluster (master + worker) | `docker-compose.yml` |
| **4** | DAGs, operators, task dependencies, parameterized DAGs, scheduling | `airflow/dags/ecommerce_dag.py` |
