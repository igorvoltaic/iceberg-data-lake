# Iceberg Data Lake with MinIO, HMS, Trino and ClickHouse

This repository provides a complete, containerized data lake platform based on **Apache Iceberg**, **MinIO** (S3‑compatible storage), **Hive Metastore** (catalog), **Trino** (for data ingestion and DDL) and **ClickHouse** (for fast analytical queries). All components are orchestrated with Docker Compose.

## Architecture

- **MinIO** – Object storage for Iceberg data and metadata (S3 API).
- **PostgreSQL** – Relational backend for the Hive Metastore.
- **Hive Metastore (HMS) v4.0.0** – Iceberg catalog service (Thrift API).
- **Trino 476** – SQL engine used to create Iceberg schemas, tables and insert sample data.
- **ClickHouse v26.3LTS** – Query engine that reads Iceberg tables directly from MinIO using the `DataLakeCatalog` engine.

All services run in a single Docker network and share persistent volumes.

## Prerequisites

- Docker Engine and Docker Compose
- `curl` (to download JARs)

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/yourusername/iceberg-data-lake.git
cd iceberg-data-lake
```

### 2. Download required JARs for Hive Metastore

The Hive Metastore image does not include the PostgreSQL JDBC driver or the Hadoop AWS libraries needed for S3 access. Download them into the `hive-lib` directory:

```bash
mkdir -p hive-lib

# PostgreSQL JDBC driver
curl -L -o hive-lib/postgresql-42.7.3.jar \
  https://jdbc.postgresql.org/download/postgresql-42.7.3.jar

# Hadoop AWS libraries (for S3A filesystem)
curl -L -o hive-lib/hadoop-aws-3.3.4.jar \
  https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-aws/3.3.4/hadoop-aws-3.3.4.jar

# AWS Java SDK bundle (dependency for Hadoop AWS)
curl -L -o hive-lib/aws-java-sdk-bundle-1.12.262.jar \
  https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk-bundle/1.12.262/aws-java-sdk-bundle-1.12.262.jar
```

### 3. Start all services

```bash
docker compose up -d
```

This starts:

| Service | Ports |
| ------- | ----- |
| MinIO (S3 API + Console) | 9000, 9001 |
| PostgreSQL | 5432 |
| Hive Metastore | 9083 |
| Trino | 8080 |
| ClickHouse | 8123 (HTTP), 9003 (native) |

### 4. Create the MinIO bucket

The Hive Metastore and Iceberg tables expect a bucket named `warehouse`. Create it using the MinIO client:

```bash
docker compose exec minio mc alias set myminio http://minio:9000 minioadmin minioadmin

docker compose exec minio mc mb myminio/warehouse
```

> Replace `iceberg-data-lake_default` with the actual network name if your project name is different (run `docker network ls` to confirm).

### 5. Populate the data lake with sample tables (using Trino)

Trino provides full DDL and DML support for Iceberg. Connect to it:

```bash
docker compose exec trino trino
```

Run the following SQL:

```sql
-- Create a schema (namespace) inside the Iceberg catalog
CREATE SCHEMA IF NOT EXISTS iceberg.demo
WITH (location = 's3a://warehouse/demo');

-- Create a customers table partitioned by country
CREATE TABLE iceberg.demo.customers (
    customer_id BIGINT,
    name VARCHAR,
    email VARCHAR,
    country VARCHAR,
    signup_date DATE,
    total_spent DECIMAL(10,2)
)
WITH (
    format = 'PARQUET',
    partitioning = ARRAY['country']
);

-- Insert customer data
INSERT INTO iceberg.demo.customers (customer_id, name, email, country, signup_date, total_spent)
VALUES
    (1, 'Alice Chen', 'alice@example.com', 'USA', DATE '2024-01-15', 1250.75),
    (2, 'Bob Martin', 'bob@example.com', 'Canada', DATE '2024-02-20', 980.50),
    (3, 'Carlos Rivera', 'carlos@example.com', 'Mexico', DATE '2024-03-10', 2340.00),
    (4, 'Diana Patel', 'diana@example.com', 'India', DATE '2024-01-05', 750.25),
    (5, 'Emma Schmidt', 'emma@example.com', 'Germany', DATE '2024-04-01', 1875.00);

-- Create an orders table partitioned by order month
CREATE TABLE iceberg.demo.orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_date TIMESTAMP,
    amount DECIMAL(10,2),
    status VARCHAR
)
WITH (
    format = 'PARQUET',
    partitioning = ARRAY['month(order_date)']
);

-- Insert order data
INSERT INTO iceberg.demo.orders (order_id, customer_id, order_date, amount, status)
VALUES
    (1001, 1, TIMESTAMP '2024-02-10 10:30:00', 299.99, 'completed'),
    (1002, 2, TIMESTAMP '2024-03-15 14:20:00', 149.50, 'completed'),
    (1003, 1, TIMESTAMP '2024-04-05 09:15:00', 899.00, 'completed'),
    (1004, 3, TIMESTAMP '2024-04-12 16:45:00', 450.75, 'pending'),
    (1005, 5, TIMESTAMP '2024-04-20 11:00:00', 1250.00, 'completed');

-- Verify the inserted data
SELECT * FROM iceberg.demo.customers;
SELECT * FROM iceberg.demo.orders;
SELECT COUNT(*) FROM iceberg.demo.customers;
```

Exit Trino (`exit` or `Ctrl+D`).

### 6. Configure ClickHouse to use the Iceberg catalog

Connect to ClickHouse:

```bash
docker compose exec clickhouse clickhouse-client
```

Then create a database that points to the Hive Metastore and MinIO:

```sql
CREATE DATABASE iceberg_lake
ENGINE = DataLakeCatalog('thrift://hms:9083', '', '')
SETTINGS
    catalog_type = 'hive',
    warehouse = 'warehouse',
    storage_endpoint = 'http://minio:9000',
    aws_access_key_id = 'minioadmin',
    aws_secret_access_key = 'minioadmin',
    region = 'us-east-1';
```

Exit the ClickHouse client with `exit`.

### 7. Query the Iceberg tables from ClickHouse

Now that the tables exist in MinIO and are registered in HMS, you can query them directly from ClickHouse.

```bash
docker compose exec clickhouse clickhouse-client
```

```sql
SELECT count() FROM iceberg_lake.`demo.customers`;
SELECT * FROM iceberg_lake.`demo.orders` LIMIT 5;
```

You should see the same data that was inserted via Trino – a single source of truth.

## Configuration files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Defines services, networks, and volumes. |
| `clickhouse-config/config.xml` | Sets S3 path‑style access for ClickHouse (required for MinIO). |
| `hive-config/hive-site.xml` | Configures HMS to use PostgreSQL and the S3A filesystem. |
| `hive-lib/` | Contains the PostgreSQL JDBC driver and Hadoop AWS libraries. |
| `trino-catalog/iceberg.properties` | Contains the Trino iceberg connector config. |

## Environment variables (defaults)

| Service | Credential |
|---------|------------|
| MinIO | `minioadmin` / `minioadmin` |
| PostgreSQL | `hive` / `hive` (database `metastore`) |
| Hive Metastore | No authentication (Thrift) |
| Clickchouse | No authentication |
| Trino | No authentication |

Change these in `docker-compose.yml` if needed.

## Troubleshooting

### HMS fails to start

- Ensure the required JARs are present in `hive-lib/` and are mounted correctly (they are, by the `docker-compose.yml`).
- If the metastore schema is not initialised, run:

```bash
docker compose run --rm hms schematool -dbType postgres -initSchema
```

### 403 Forbidden when querying Iceberg tables from ClickHouse

- Verify the bucket `warehouse` exists in MinIO.
- Check that `storage_endpoint` is `http://minio:9000` (no bucket name).
- Confirm that path‑style access is enabled in `clickhouse-config/config.xml` (the file provided does this).
- Ensure the ClickHouse container has `AWS_EC2_METADATA_DISABLED=true` (set in `docker-compose.yml`).

### Trino cannot see the Iceberg catalog

- Verify MinIO is healthy (`docker compose ps`).
- Check that the `warehouse` bucket exists and that the `demo/` directory has been created.
- Ensure the Iceberg connector in Trino is configured to use `thrift://hms:9083` (the default configuration is provided).

## Stopping the stack

```bash
docker compose down -v   # removes all data volumes
```

## References

- [Apache Iceberg](https://iceberg.apache.org/)
- [ClickHouse DataLakeCatalog](https://clickhouse.com/docs/engines/database-engines/datalakecatalog)
- [MinIO](https://min.io/)
- [Trino Iceberg connector](https://trino.io/docs/476/connector/iceberg.html)
- [Apache Hive Metastore](https://cwiki.apache.org/confluence/display/Hive/Design)

## License

MIT
