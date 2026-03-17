# Day 18 Lab: Amazon Redshift — Data Warehousing

**Type:** Attendee Lab  
**Duration:** ~90 minutes  
**Day:** 18 — Data Warehousing with Amazon Redshift  
**Your Prefix:** `traineeNN` (replace `NN` with your number)

---

## Lab Setup

### Configure your AWS CLI

> **Already done on Day 15?** If your profile is still active, just verify it:
> ```bash
> export AWS_PROFILE=aws-de-lab
> aws sts get-caller-identity
> ```
> If this returns your `$PREFIX` ARN, skip to Part 1. If not, re-run `aws configure` below.

In your workstation terminal (or local terminal):

```bash
aws configure --profile aws-de-lab
```

Enter the values from your credential sheet:

```
AWS Access Key ID:     <your-access-key-id>
AWS Secret Access Key: <your-secret-access-key>
Default region name:   ap-south-1
Default output format: json
```

Verify your identity:

```bash
export AWS_PROFILE=aws-de-lab
aws sts get-caller-identity
```

Expected output:

```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "<ACCOUNT_ID>",
    "Arn": "arn:aws:iam::<ACCOUNT_ID>:user/$PREFIX"
}
```

> Replace `traineeNN` with your actual attendee ID throughout this lab.

Set your prefix once — all commands below use `$PREFIX` and `$BATCH`:

```bash
export PREFIX=traineeNN   # ← replace traineeNN with your ID, e.g. trainee07
export BATCH=2026-03
export CATALOG="${PREFIX}_${BATCH//-/_}_catalog"   # Glue catalog DB name
export AWS_PROFILE=aws-de-lab
export REGION=ap-south-1
```

---

## What You Will Build

By the end of this lab you will:

1. Connect to your Redshift cluster using both the Query Editor v2 and `psql`
2. Load CSV data from S3 using the `COPY` command
3. Apply Redshift-specific table design (distribution keys, sort keys)
4. Run analytical queries and inspect query execution plans
5. Observe how `VACUUM` and `ANALYZE` maintain cluster performance
6. Query S3 directly via Redshift Spectrum

---

## Prerequisites

- Day 17 lab complete (your RDS PostgreSQL instance has data)
- `psql` client installed (same as Day 17)
- AWS CLI configured (`aws configure list` shows your profile)

---

## Part 1: Retrieve Your Redshift Credentials (10 min)

Your Redshift cluster credentials are stored in AWS Secrets Manager, separate from your RDS credentials.

### Retrieve via console

1. Open **AWS Console** → **Secrets Manager**
2. Search for: `aws-de-lab/$BATCH/$PREFIX/redshift`
3. Click the secret → **Retrieve secret value**
4. Note the values:

| Field | Your value |
|---|---|
| `host` | `$PREFIX-$BATCH-redshift.xxxxxxxxx.ap-south-1.redshift.amazonaws.com` |
| `port` | `5439` |
| `dbname` | `labdb` |
| `username` | `labadmin` |
| `password` | (copy this) |

### Retrieve via CLI

```bash
SECRET=$(aws secretsmanager get-secret-value \
  --secret-id "aws-de-lab/$BATCH/$PREFIX/redshift" \
  --query SecretString --output text)

export REDSHIFT_HOST=$(echo $SECRET | python3 -c "import sys,json; print(json.load(sys.stdin)['host'])")
export REDSHIFT_PORT=$(echo $SECRET | python3 -c "import sys,json; print(json.load(sys.stdin)['port'])")
export REDSHIFT_DB=$(echo $SECRET | python3 -c "import sys,json; print(json.load(sys.stdin)['dbname'])")
export REDSHIFT_USER=$(echo $SECRET | python3 -c "import sys,json; print(json.load(sys.stdin)['username'])")
export PGPASSWORD=$(echo $SECRET | python3 -c "import sys,json; print(json.load(sys.stdin)['password'])")
```

### Verify your cluster details

```bash
aws redshift describe-clusters \
  --cluster-identifier $PREFIX-$BATCH-redshift \
  --query 'Clusters[0].{Status:ClusterStatus,NodeType:NodeType,NumberOfNodes:NumberOfNodes,DBName:DBName,Endpoint:Endpoint}'
```

Expected output:
```json
{
    "Status": "available",
    "NodeType": "ra3.large",
    "NumberOfNodes": 1,
    "DBName": "labdb",
    "Endpoint": {
        "Address": "$PREFIX-$BATCH-redshift.xxxx.ap-south-1.redshift.amazonaws.com",
        "Port": 5439
    }
}
```

---

## Part 2: Connect to Redshift via Query Editor v2 (10 min)

Query Editor v2 is the browser-based SQL editor for Redshift — no local client required.

> **Why QEv2 and psql use different users**  
> Query Editor v2 authenticates via **IAM** (`GetClusterCredentialsWithIAM`), which creates a
> separate Redshift database user named `IAMR:$PREFIX`. Your psql sessions connect
> as `labadmin` (the master user). These are two different DB users.
>
> The lab cluster's `init.sql` grants `PUBLIC` access to both `trainee_schema` and `public`,
> so tables you create as `labadmin` (via psql) **are** visible in QEv2. If the left-panel
> schema browser doesn't immediately refresh, click the **↻ Refresh** icon next to the
> schema name.

### Connect via console

1. Open **AWS Console** → **Amazon Redshift**
2. Left sidebar → **Query Editor v2**
3. Click **+** (Connect to database)
4. Connection settings:
   - **Cluster:** `$PREFIX-$BATCH-redshift`
   - **Database:** `labdb`
   - **Authentication:** `Temporary credentials` (uses your IAM identity)
5. Click **Create connection**

### Run a quick test

```sql
-- Test connectivity
SELECT version();

-- Check current user and database
SELECT current_user, current_database();
```

> Note: Redshift is PostgreSQL-compatible but **not identical**. Some PostgreSQL functions and data types behave differently.

---

## Part 3: Connect via psql (5 min)

For scripting and bulk operations, the CLI approach is faster.

```bash
# Connect using environment variables (set in Part 1)
psql -h $REDSHIFT_HOST -p $REDSHIFT_PORT -d $REDSHIFT_DB -U $REDSHIFT_USER
```

Expected prompt: `labdb=#`

```sql
-- Quick sanity check
SELECT current_user, current_database(), version();
\q
```

---

## Part 4: Design Your Table with Distribution and Sort Keys (15 min)

Redshift is a **columnar, massively parallel** database. How you design your tables determines query speed.

### Core design concepts

**Distribution style:** controls how rows are spread across compute nodes

| Style | Use when |
|---|---|
| `DISTSTYLE ALL` | Small dimension/reference tables (< 3 rows) — full copy on every node |
| `DISTSTYLE EVEN` | No clear join key, balanced parallelism |
| `DISTKEY(column)` | Large fact tables — co-locate rows for joins on this column |
| `DISTSTYLE AUTO` | Let Redshift choose (default for new tables) |

**Sort key:** controls how rows are stored on disk (like a clustered index)

| Type | Use when |
|---|---|
| `SORTKEY(column)` | Compound sort key — range scans on first column, then second, etc. |
| `INTERLEAVED SORTKEY(col1, col2)` | Each column equally important in filters |

### Create a dimension table (small, distributed to all nodes)

Open psql or Query Editor v2:

```sql
-- Small reference table: one copy on every node
CREATE TABLE vendors (
    vendor_id     VARCHAR(3)     NOT NULL,
    vendor_name   VARCHAR(100)   NOT NULL,
    PRIMARY KEY (vendor_id)
)
DISTSTYLE ALL;

INSERT INTO vendors VALUES
    ('CMT', 'Creative Mobile Technologies'),
    ('VTS', 'VeriFone Inc.');

SELECT * FROM vendors;
```

### Create a fact table (large, distributed and sorted)

```sql
-- Fact table: distribute on vendor_id so taxi data and vendors co-locate for joins
-- Sort on pickup_datetime so time-range queries skip irrelevant blocks
CREATE TABLE taxi_trips (
    vendor_id         VARCHAR(3),
    pickup_datetime   TIMESTAMP    ENCODE ZSTD,
    dropoff_datetime  TIMESTAMP    ENCODE ZSTD,
    passenger_count   SMALLINT,
    trip_distance     DECIMAL(8,2),
    pickup_lon        DECIMAL(9,6),
    pickup_lat        DECIMAL(9,6),
    dropoff_lon       DECIMAL(9,6),
    dropoff_lat       DECIMAL(9,6),
    payment_type      VARCHAR(20)  ENCODE BYTEDICT,
    fare_amount       DECIMAL(8,2),
    total_amount      DECIMAL(8,2)
)
DISTKEY(vendor_id)
SORTKEY(pickup_datetime)
;
```

> **Column encoding (ENCODE):** Redshift compresses each column independently. `ZSTD` works well for timestamps/numerics. `BYTEDICT` excels for low-cardinality strings like `payment_type`.

### Check table design

```sql
-- View distribution and sort key metadata
SELECT "table", diststyle, sortkey1
FROM SVV_TABLE_INFO
WHERE "table" IN ('taxi_trips', 'vendors');
```

---

## Part 5: Load Data from S3 with COPY (15 min)

The `COPY` command is Redshift's primary bulk-load mechanism. It loads data in parallel from S3, much faster than individual INSERT statements.

### Find your IAM role ARN

```bash
# Your Redshift cluster has an attached IAM role for S3 access
aws redshift describe-clusters \
  --cluster-identifier $PREFIX-$BATCH-redshift \
  --query 'Clusters[0].IamRoles[*].IamRoleArn'
```

Make note of the ARN — it will look like:  
`arn:aws:iam::<ACCOUNT_ID>:role/$PREFIX-$BATCH-redshift-s3-role`

### Load CSV data

In psql or Query Editor v2, run (replace `<ACCOUNT_ID>` and `NN`):

```sql
COPY taxi_trips
FROM 's3://$PREFIX-$BATCH-raw/datasets/taxi_trips_sample.csv'
IAM_ROLE 'arn:aws:iam::<ACCOUNT_ID>:role/$PREFIX-$BATCH-redshift-s3-role'
CSV
IGNOREHEADER 1
DATEFORMAT 'auto'
TIMEFORMAT 'auto'
ACCEPTINVCHARS '?';
```

> **ACCEPTINVCHARS:** Replaces invalid UTF-8 characters instead of failing the entire load — useful for real-world messy data.

### Verify the load

```sql
-- Row count
SELECT COUNT(*) FROM taxi_trips;

-- Check for load errors (if COPY failed partially)
SELECT starttime, filename, line_number, colname, err_reason
FROM STL_LOAD_ERRORS
ORDER BY starttime DESC
LIMIT 10;
```

### Check load performance via system tables

```sql
-- COPY statistics
SELECT query, rows, bytes, loadtime
FROM STL_LOAD_COMMITS
ORDER BY starttime DESC
LIMIT 5;
```

---

## Part 6: Analytical Queries (15 min)

### Basic aggregations

```sql
-- Revenue by payment type
SELECT
    payment_type,
    COUNT(*)                            AS trip_count,
    ROUND(AVG(fare_amount), 2)          AS avg_fare,
    ROUND(SUM(total_amount), 2)         AS total_revenue
FROM taxi_trips
GROUP BY payment_type
ORDER BY total_revenue DESC;
```

```sql
-- Trips by hour of day (useful for capacity planning)
SELECT
    EXTRACT(HOUR FROM pickup_datetime)  AS hour_of_day,
    COUNT(*)                            AS trip_count,
    ROUND(AVG(trip_distance), 2)        AS avg_distance
FROM taxi_trips
GROUP BY 1
ORDER BY 1;
```

```sql
-- Join with dimension table (uses co-location from DISTKEY)
SELECT
    v.vendor_name,
    COUNT(*)                            AS trips,
    ROUND(SUM(t.total_amount), 2)       AS revenue
FROM taxi_trips t
JOIN vendors v ON t.vendor_id = v.vendor_id
GROUP BY v.vendor_name
ORDER BY revenue DESC;
```

### Examine the query execution plan

```sql
-- EXPLAIN shows the query plan without executing
EXPLAIN
SELECT
    v.vendor_name,
    COUNT(*) AS trips
FROM taxi_trips t
JOIN vendors v ON t.vendor_id = v.vendor_id
GROUP BY v.vendor_name;
```

Look for:
- **DS_DIST_NONE** — join does NOT require data redistribution ✅ (DISTKEY worked)
- **DS_DIST_INNER** — inner table is broadcast to all nodes (used for `DISTSTYLE ALL`)
- **DS_BCAST_INNER** — small table broadcast for a join
- **DS_DIST_BOTH** — both tables need redistribution ❌ (design mismatch)

---

## Part 7: VACUUM and ANALYZE (10 min)

### Why VACUUM is needed in Redshift

Redshift uses an **append-only** storage model with marked-deletion. DELETE statements mark rows as invisible but don't free space. UPDATE = DELETE old row + INSERT new row.

```sql
-- Insert some test rows
INSERT INTO taxi_trips
SELECT * FROM taxi_trips;   -- doubles the data

-- Delete half the rows
DELETE FROM taxi_trips
WHERE EXTRACT(MINUTE FROM pickup_datetime) > 30;

-- Check unsorted and deleted row percentages
SELECT
    "table",
    pct_used,
    unsorted,
    stats_off
FROM svv_table_info
WHERE "table" = 'taxi_trips';
```

### Run VACUUM

```sql
-- VACUUM reclaims deleted space and re-sorts rows according to SORTKEY
VACUUM taxi_trips;
```

> In production, use `VACUUM DELETE ONLY` or `VACUUM SORT ONLY` to limit scope and reduce I/O.

### Run ANALYZE

```sql
-- ANALYZE updates table statistics for the query planner
ANALYZE taxi_trips;

-- Confirm stats_off is now 0
SELECT "table", stats_off
FROM svv_table_info
WHERE "table" = 'taxi_trips';
```

> Redshift also runs automatic VACUUM and ANALYZE in the background. Manual runs are needed after large bulk loads.

---

## Part 8: Redshift Spectrum — Query S3 Directly (10 min)

Redshift Spectrum lets you query data in S3 **without loading it** into Redshift storage. Billing is per-byte scanned (like Athena).

### Create an external schema pointing to your Glue catalog

```sql
-- External schema links Redshift to the Glue Data Catalog
-- This database was created by your Glue crawler on Day 19
-- Replace <ACCOUNT_ID> and NN
CREATE EXTERNAL SCHEMA IF NOT EXISTS spectrum_raw
FROM DATA CATALOG
DATABASE '$CATALOG'
IAM_ROLE 'arn:aws:iam::<ACCOUNT_ID>:role/$PREFIX-$BATCH-redshift-s3-role'
REGION 'ap-south-1';
```

> **Note:** The Glue catalog database uses underscores: `$CATALOG`  
> (Glue catalog database names cannot contain hyphens)

### Query S3 through Spectrum

After Day 19 (Glue) runs the crawler, you will be able to run:

```sql
-- List tables in the external schema
SELECT schemaname, tablename
FROM SVV_EXTERNAL_TABLES
WHERE schemaname = 'spectrum_raw';

-- Query raw CSV directly from S3 (zero bytes loaded to Redshift)
SELECT
    vendor_id,
    COUNT(*)            AS trip_count,
    SUM(total_amount)   AS total_revenue
FROM spectrum_raw.taxi_trips   -- this reads from S3, not Redshift storage
GROUP BY vendor_id;
```

### Compare: Redshift local vs Spectrum

```sql
-- Local Redshift table (fast, uses compressed columnar storage)
SELECT COUNT(*) FROM public.taxi_trips;

-- Spectrum (S3 scan, charged per byte — great for archival/infrequent data)
SELECT COUNT(*) FROM spectrum_raw.taxi_trips;

-- In practice, use Spectrum for data older than 1 year
-- Keep recent/hot data in Redshift local storage
```

---

## Validation Checklist

Before ending the lab, verify:

```bash
# 1. Table exists with correct design
psql -h $REDSHIFT_HOST -p $REDSHIFT_PORT -d $REDSHIFT_DB -U $REDSHIFT_USER \
  -c "SELECT \"table\", diststyle, sortkey1 FROM SVV_TABLE_INFO WHERE \"table\" = 'taxi_trips';"

# 2. Data was loaded
psql -h $REDSHIFT_HOST -p $REDSHIFT_PORT -d $REDSHIFT_DB -U $REDSHIFT_USER \
  -c "SELECT COUNT(*) FROM taxi_trips;"

# 3. No COPY errors
psql -h $REDSHIFT_HOST -p $REDSHIFT_PORT -d $REDSHIFT_DB -U $REDSHIFT_USER \
  -c "SELECT COUNT(*) FROM stl_load_errors WHERE starttime > current_timestamp - interval '2 hours';"
```

Expected: table exists with `DISTKEY` and `SORTKEY`, row count > 0, zero load errors.

---

## Redshift vs PostgreSQL — Key Differences

| Feature | PostgreSQL (Day 17) | Amazon Redshift |
|---|---|---|
| **Purpose** | OLTP — many small transactions | OLAP — large analytical scans |
| **Storage** | Row-oriented | Columnar |
| **Indexes** | B-tree, GIN, BRIN | Sort keys (zone maps), no traditional indexes |
| **Concurrency** | Thousands of sessions | Tens to hundreds of sessions |
| **Port** | 5432 | 5439 |
| **VACUUM needed?** | Autovacuum handles it | Manual + background auto-vacuum |
| **COPY command?** | Via `COPY … FROM STDIN` (client-side) | `COPY … FROM 's3://…'` (server-side parallel) |
| **Sequences** | Supported fully | `IDENTITY` column preferred |
| **Stored procedures** | PL/pgSQL | PL/pgSQL (limited compatibility) |

---

*Next: [Instructor Demo — Redshift Advanced](demo-redshift-advanced.md)*  
*Day 19: [Glue + Athena Lab](../day19/lab-glue-athena.md)*
