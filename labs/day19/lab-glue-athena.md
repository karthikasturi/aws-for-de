# Day 19 Lab: AWS Glue + Amazon Athena — Serverless ETL and Query

**Type:** Attendee Lab  
**Duration:** ~90 minutes  
**Day:** 19 — Serverless ETL and Queries  
**Your Prefix:** set once as `$TRAINEE` in Lab Setup — all bash commands use it automatically

---

## Lab Setup

### Configure your AWS CLI

> **Already done on Day 15?** If your profile is still active, just verify it:
> ```bash
> export AWS_PROFILE=aws-de-lab
> aws sts get-caller-identity
> ```
> If this returns your `traineeNN` ARN, skip to Part 1. If not, re-run `aws configure` below.

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

Verify your identity and set your trainee ID:

```bash
export AWS_PROFILE=aws-de-lab
aws sts get-caller-identity
```

Expected output:

```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "<ACCOUNT_ID>",
    "Arn": "arn:aws:iam::<ACCOUNT_ID>:user/traineeNN"
}
```

Set your trainee ID once — all bash commands in this lab reference `$TRAINEE` automatically:

```bash
export TRAINEE=traineeNN   # ← change traineeNN to your ID, e.g. trainee07
echo "Trainee ID: $TRAINEE"
```

> The **Athena console SQL editor** runs in the browser and cannot read shell variables. For SQL blocks marked **console**, replace `traineeNN` with your actual ID manually.

---

## What You Will Build

By the end of this lab you will:

1. Explore the pre-provisioned Glue infrastructure (catalog, connection, crawler, ETL job)
2. Run the Glue crawler to discover raw CSV data in S3
3. Execute the pre-written PySpark ETL job to convert CSV → Parquet
4. Query both the raw and transformed data using Amazon Athena
5. Understand how cost controls protect the lab (100 MB per-query scan limit)

---

## Architecture Overview

```
S3 Raw Bucket                  Glue                      S3 Transformed Bucket
traineeNN-2026-03-raw       ───────────────────         traineeNN-2026-03-transformed
  datasets/                    Crawler                       taxi_parquet/
    taxi_trips_sample.csv  ───►  (schema discovery)    ╭──►   pickup_year=YYYY/
  taxi/                             │                   │       pickup_month=MM/
    *.csv (future uploads)          ▼                   │         *.parquet
                             Glue Data Catalog          │
                               traineeNN_               │   Athena Workgroup
                               2026-03_catalog   ────┤   traineeNN-2026-03-wg
                                 taxi_trips     ◄──╮    │     Results:
                                 (CSV schema)      │    │   traineeNN-2026-03-
                                                   │    │   athena-output/results/
                             ETL Job               │    │
                               traineeNN-          │    │
                               2026-03-etl-job  │    │
                               (PySpark)  ─────────────►╯
```

---

## Prerequisites

- AWS CLI configured (verify: `aws configure list`)
- Day 16's taxi CSV data is in your raw bucket (verify below)

```bash
# Check your raw data exists
aws s3 ls s3://${TRAINEE}-2026-03-raw/datasets/
```

Expected: `taxi_trips_sample.csv` is present.

---

## Part 1: Explore Glue Infrastructure (15 min)

### 1.1 View your Glue Data Catalog database

```bash
# List existing catalog databases
aws glue get-databases \
  --query 'DatabaseList[*].Name' --output text | grep "^${TRAINEE}"
```

Navigate to console: **AWS Glue** → **Data Catalog** → **Databases**

Your catalog database is named with **underscores** (not hyphens — Glue catalog requires this):
```
${TRAINEE}_2026_03_catalog
```

```bash
# Inspect the database
aws glue get-database \
  --name "${TRAINEE}_2026_03_catalog" \
  --query 'Database.{Name:Name,Description:Description,LocationUri:LocationUri}'
```

At this point the database is empty — the crawler hasn't run yet. You'll see tables after Part 2.

### 1.2 Inspect your Glue crawler

```bash
# List your crawlers
aws glue list-crawlers \
  --query 'CrawlerNames' --output text | grep "^${TRAINEE}"

# Show crawler configuration
aws glue get-crawler \
  --name "${TRAINEE}-2026-03-raw-crawler" \
  --query 'Crawler.{State:State,Targets:Targets,DatabaseName:DatabaseName,Schedule:Schedule}'
```

Key configuration to note:
- **S3 target path:** `s3://${TRAINEE}-2026-03-raw/`
- **Target database:** `${TRAINEE}_2026_03_catalog`
- **Schedule:** no schedule (on-demand only — run manually)
- **State:** READY

### 1.3 Inspect your Glue ETL job

```bash
# Show job configuration
aws glue get-job \
  --job-name "${TRAINEE}-2026-03-etl-job" \
  --query 'Job.{Name:Name,Role:Role,Timeout:Timeout,GlueVersion:GlueVersion,WorkerType:WorkerType,NumberOfWorkers:NumberOfWorkers,ScriptLocation:Command.ScriptLocation}'
```

Expected output:
```json
{
    "Name": "traineeNN-2026-03-etl-job",
    "Timeout": 30,
    "GlueVersion": "4.0",
    "WorkerType": "G.1X",
    "NumberOfWorkers": 2,
    "ScriptLocation": "s3://traineeNN-2026-03-scripts/glue/glue_etl_job.py"
}
```

> **Timeout = 30:** Not the AWS default (2880 minutes = 48 hours!). The lab enforces 30 minutes maximum to protect your sandbox budget.

### 1.4 Download and read the ETL script

```bash
# Download the script so you can inspect it
aws s3 cp \
  s3://${TRAINEE}-2026-03-scripts/glue/glue_etl_job.py \
  ~/glue_etl_job.py

cat ~/glue_etl_job.py
```

Key things to observe in the script:
- It accepts `--SOURCE_BUCKET`, `--TARGET_BUCKET`, `--DATABASE_NAME` as job arguments
- Reads `s3://<SOURCE_BUCKET>/taxi/*.csv` (note: `taxi/` prefix, not `datasets/`)
- Writes Parquet to `s3://<TARGET_BUCKET>/taxi_parquet/`, partitioned by `pickup_year` / `pickup_month`
- Registers the output as table `taxi_trips_parquet` in the Glue catalog
- Uses Glue 4.0 (Spark 3.3, Python 3.10)

### 1.5 Inspect your Athena workgroup

```bash
# View workgroup settings
aws athena get-work-group \
  --work-group "${TRAINEE}-2026-03-wg" \
  --query 'WorkGroup.{State:State,BytesScannedCutoffPerQuery:Configuration.BytesScannedCutoffPerQuery,ResultLocation:Configuration.ResultConfiguration.OutputLocation}'
```

Expected output:
```json
{
    "State": "ENABLED",
    "BytesScannedCutoffPerQuery": 104857600,
    "ResultLocation": "s3://traineeNN-2026-03-athena-output/results/"
}
```

> **104857600 bytes = 100 MB.** Any Athena query that would scan more than 100 MB will be cancelled before running. This prevents accidental full-table scans on large datasets from burning budget.

---

## Part 2: Run the Glue Crawler (10 min)

The crawler inspects S3 and infers table schemas (column names, data types, partitions) automatically.

### 2.1 Copy data to the taxi/ prefix that the ETL job expects

The ETL script reads from `s3://<SOURCE_BUCKET>/taxi/*.csv`, so copy the sample data:

```bash
aws s3 cp \
  s3://${TRAINEE}-2026-03-raw/datasets/taxi_trips_sample.csv \
  s3://${TRAINEE}-2026-03-raw/taxi/taxi_trips_sample.csv
```

### 2.2 Start the crawler

```bash
aws glue start-crawler --name "${TRAINEE}-2026-03-raw-crawler"
echo "Crawler started. Polling status..."

# Poll until complete (typically 1-2 minutes)
while true; do
  STATUS=$(aws glue get-crawler \
    --name "${TRAINEE}-2026-03-raw-crawler" \
    --query 'Crawler.State' --output text)
  echo "$(date '+%H:%M:%S') Status: $STATUS"
  [[ "$STATUS" == "READY" ]] && break
  sleep 15
done

echo "Crawler finished."
```

### 2.3 View the discovered tables

```bash
# List tables created in the catalog
aws glue get-tables \
  --database-name "${TRAINEE}_2026_03_catalog" \
  --query 'TableList[*].{Name:Name,InputFormat:StorageDescriptor.InputFormat,Location:StorageDescriptor.Location}'
```

You should see a `taxi` table (named after the S3 folder prefix).

```bash
# View the full schema of the taxi table
aws glue get-table \
  --database-name "${TRAINEE}_2026_03_catalog" \
  --name taxi \
  --query 'Table.StorageDescriptor.Columns[*]'
```

Expected columns: `vendor_id`, `pickup_datetime`, `dropoff_datetime`, `passenger_count`, `trip_distance`, `pickup_lon`, `pickup_lat`, `dropoff_lon`, `dropoff_lat`, `payment_type`, `fare_amount`, `total_amount`.

> If the crawler named it differently (e.g. `taxi_trips_sample`), use `aws glue get-tables` to see the exact table name.

---

## Part 3: Query Raw CSV Data with Athena (10 min)

Before transforming, query the raw CSV directly through Athena to confirm the schema.

### 3.1 Run a query via CLI

```bash
# Start a query (note: must use your workgroup to enforce the 100 MB limit)
QUERY_ID=$(aws athena start-query-execution \
  --query-string "SELECT * FROM \"${TRAINEE}_2026_03_catalog\".taxi LIMIT 5;" \
  --work-group "${TRAINEE}-2026-03-wg" \
  --query 'QueryExecutionId' --output text)

echo "Query ID: $QUERY_ID"

# Wait for completion
aws athena wait query-execution-complete \
  --query-execution-id $QUERY_ID

# Get results
aws athena get-query-results \
  --query-execution-id $QUERY_ID \
  --query 'ResultSet.Rows[*].Data[*].VarCharValue'
```

### 3.2 Query via Athena console

> **Your resources** (run in terminal to confirm): `echo "WG: ${TRAINEE}-2026-03-wg  DB: ${TRAINEE}_2026_03_catalog"`

1. Open **AWS Console** → **Amazon Athena**
2. **Settings** → verify output location is `s3://traineeNN-2026-03-athena-output/results/`
3. Change **Workgroup** to `traineeNN-2026-03-wg` (top-right workgroup selector)
4. In the **Query editor**, select database `traineeNN_2026_03_catalog`
5. Run (replace `traineeNN` with your ID):

```sql
-- Preview raw CSV data
SELECT
    vendor_id,
    pickup_datetime,
    trip_distance,
    fare_amount
FROM taxi
LIMIT 10;
```

```sql
-- Data types check — Athena inferred these from CSV
DESCRIBE taxi;
```

> Note: Athena reads the CSV fresh from S3 on every query — there's no caching of the file itself. CSV is slow for analytical queries because it's row-oriented and uncompressed.

---

## Part 4: Run the Glue ETL Job (15 min)

The ETL job converts the CSV data to **Parquet** — a columnar, compressed format that Athena queries much faster (and scans fewer bytes = lower cost).

### 4.1 Start the ETL job

```bash
# Start the ETL job with three custom arguments
JOB_RUN_ID=$(aws glue start-job-run \
  --job-name "${TRAINEE}-2026-03-etl-job" \
  --arguments "{
    \"--SOURCE_BUCKET\": \"${TRAINEE}-2026-03-raw\",
    \"--TARGET_BUCKET\": \"${TRAINEE}-2026-03-transformed\",
    \"--DATABASE_NAME\": \"${TRAINEE}_2026_03_catalog\"
  }" \
  --query 'JobRunId' --output text)

echo "Job Run ID: $JOB_RUN_ID"
```

### 4.2 Monitor the job

```bash
# Poll until complete (typically 3-5 minutes for 2 G.1X workers)
while true; do
  STATE=$(aws glue get-job-run \
    --job-name "${TRAINEE}-2026-03-etl-job" \
    --run-id $JOB_RUN_ID \
    --query 'JobRun.JobRunState' --output text)

  ELAPSED=$(aws glue get-job-run \
    --job-name "${TRAINEE}-2026-03-etl-job" \
    --run-id $JOB_RUN_ID \
    --query 'JobRun.ExecutionTime' --output text 2>/dev/null || echo "N/A")

  echo "$(date '+%H:%M:%S') State: $STATE | Elapsed: ${ELAPSED}s"

  [[ "$STATE" == "SUCCEEDED" || "$STATE" == "FAILED" || "$STATE" == "STOPPED" ]] && break
  sleep 20
done
```

### 4.3 Check for errors if the job fails

```bash
# View error message if failed
aws glue get-job-run \
  --job-name "${TRAINEE}-2026-03-etl-job" \
  --run-id $JOB_RUN_ID \
  --query 'JobRun.ErrorMessage'

# View CloudWatch logs
LOG_GROUP="/aws/glue/${TRAINEE}-2026-03-jobs"
LOG_STREAM=$(aws logs describe-log-streams \
  --log-group-name $LOG_GROUP \
  --order-by LastEventTime \
  --descending \
  --query 'logStreams[0].logStreamName' --output text)

aws logs get-log-events \
  --log-group-name $LOG_GROUP \
  --log-stream-name $LOG_STREAM \
  --limit 50 \
  --query 'events[*].message' --output text
```

### 4.4 Verify the Parquet output

```bash
# Check transformed data in S3
aws s3 ls s3://${TRAINEE}-2026-03-transformed/taxi_parquet/ --recursive

# Should see partition structure:
# taxi_parquet/pickup_year=YYYY/pickup_month=MM/*.parquet
```

### 4.5 Verify the new Glue catalog table

```bash
aws glue get-table \
  --database-name "${TRAINEE}_2026_03_catalog" \
  --name taxi_trips_parquet \
  --query 'Table.{Name:Name,Format:StorageDescriptor.InputFormat,Location:StorageDescriptor.Location,Partitions:PartitionKeys[*].Name}'
```

Expected: `InputFormat` contains `parquet`, `Partitions` = `["pickup_year", "pickup_month"]`.

---

## Part 5: Query Parquet Data with Athena (20 min)

Now query the transformed Parquet data and compare with the CSV query.

### 5.1 Run aggregation queries

In the Athena console with workgroup `traineeNN-2026-03-wg`:

> **Console reminder:** replace `traineeNN` in the SQL below with your actual ID. Run `echo $TRAINEE` in your terminal to confirm.

```sql
-- Revenue summary by payment type on Parquet (much faster scan)
-- console: replace traineeNN with your ID
SELECT
    payment_type,
    COUNT(*)                                    AS trip_count,
    ROUND(AVG(CAST(fare_amount AS DOUBLE)), 2)  AS avg_fare,
    ROUND(SUM(CAST(total_amount AS DOUBLE)), 2) AS total_revenue
FROM "traineeNN_2026_03_catalog"."taxi_trips_parquet"
GROUP BY payment_type
ORDER BY total_revenue DESC;
```

```sql
-- Trips per hour of day (useful for airport/rideshare capacity planning)
-- console: replace traineeNN with your ID
SELECT
    EXTRACT(HOUR FROM CAST(pickup_datetime AS TIMESTAMP)) AS pickup_hour,
    COUNT(*)                                                AS trips
FROM "traineeNN_2026_03_catalog"."taxi_trips_parquet"
GROUP BY 1
ORDER BY 1;
```

```sql
-- Partition pruning in action — Athena only scans the matching partition
-- console: replace traineeNN with your ID
SELECT COUNT(*)
FROM "traineeNN_2026_03_catalog"."taxi_trips_parquet"
WHERE pickup_year = '2024'
  AND pickup_month = '01';
```

### 5.2 Check scan statistics (key for cost awareness)

After each query, check the query execution details in Athena console:
- Click on the query history tab → Last query
- Note **Data scanned** in the results panel

```sql
-- Compare scan cost: CSV vs Parquet (console: replace traineeNN with your ID)
-- 1. First query CSV (note Data scanned value)
SELECT COUNT(*) FROM "traineeNN_2026_03_catalog"."taxi";

-- 2. Then query Parquet (note Data scanned value)
SELECT COUNT(*) FROM "traineeNN_2026_03_catalog"."taxi_trips_parquet";
```

> Parquet typically scans **10–100× fewer bytes** than CSV for the same query — because:
> 1. Columnar format: only reads columns you SELECT, not the entire row
> 2. Statistics per column group (row groups): can skip entire blocks
> 3. Compression: SNAPPY or ZSTD reduces I/O

### 5.3 Create a named Athena saved query

Console → **Saved queries** → **Create saved query**:

```sql
-- Saved query: daily_revenue_summary (save with this name)
-- console: replace traineeNN with your ID
SELECT
    CAST(pickup_datetime AS DATE)                           AS trip_date,
    SUM(CAST(total_amount AS DOUBLE))                       AS daily_revenue,
    COUNT(*)                                                AS trip_count,
    ROUND(AVG(CAST(trip_distance AS DOUBLE)), 2)            AS avg_distance
FROM "traineeNN_2026_03_catalog"."taxi_trips_parquet"
GROUP BY 1
ORDER BY 1;
```

### 5.4 Use the CLI for programmatic queries

```bash
# Run a query and download results as CSV
QUERY_ID=$(aws athena start-query-execution \
  --query-string "SELECT vendor_id, COUNT(*) AS trips, SUM(CAST(total_amount AS DOUBLE)) AS revenue FROM \"${TRAINEE}_2026_03_catalog\".\"taxi_trips_parquet\" GROUP BY vendor_id ORDER BY vendor_id;" \
  --work-group "${TRAINEE}-2026-03-wg" \
  --query 'QueryExecutionId' --output text)

aws athena wait query-execution-complete --query-execution-id $QUERY_ID

# Download the result CSV from S3
RESULT_FILE=$(aws athena get-query-execution \
  --query-execution-id $QUERY_ID \
  --query 'QueryExecution.ResultConfiguration.OutputLocation' --output text)

aws s3 cp "$RESULT_FILE" ~/athena_result.csv
cat ~/athena_result.csv
```

---

## Part 6: Glue Studio Visual Overview (5 min)

The Glue ETL job in this lab is a PySpark script, but Glue also offers a visual ETL editor.

### View your job in Glue Studio

1. Console → **AWS Glue** → **ETL Jobs**
2. Click `traineeNN-2026-03-etl-job` (run `echo "${TRAINEE}-2026-03-etl-job"` to confirm your job name)
3. Note: This job was created as a **Script** type. Click **Script** tab to see the PySpark code
4. Click **Runs** tab → verify your job run history with `SUCCEEDED` status
5. Click on the successful run → **Output logs** → click the CloudWatch link to see Spark driver output

---

## Validation Checklist

```bash
# 1. Crawler created a table in Glue catalog
aws glue get-tables \
  --database-name "${TRAINEE}_2026_03_catalog" \
  --query 'TableList[*].Name'

# 2. ETL job run succeeded
aws glue get-job-runs \
  --job-name "${TRAINEE}-2026-03-etl-job" \
  --query 'JobRuns[0].{State:JobRunState,Duration:ExecutionTime}' \
  --max-results 1

# 3. Parquet files exist in transformed bucket
aws s3 ls s3://${TRAINEE}-2026-03-transformed/taxi_parquet/ --recursive | head -5

# 4. Athena can query the Parquet table
aws athena start-query-execution \
  --query-string "SELECT COUNT(*) FROM \"${TRAINEE}_2026_03_catalog\".\"taxi_trips_parquet\";" \
  --work-group "${TRAINEE}-2026-03-wg" \
  --query 'QueryExecutionId' --output text
```

---

## Glue vs Athena — When to Use Each

| Scenario | Use |
|---|---|
| Convert CSV to Parquet at scale | **Glue ETL** (PySpark) |
| Apply complex transformations (dedup, join, ML feature prep) | **Glue ETL** |
| Query data in S3 ad-hoc without moving it | **Athena** |
| Power a BI dashboard against S3 data | **Athena** |
| Auto-discover schemas from new S3 data | **Glue Crawler** |
| Store and version the schema | **Glue Data Catalog** |
| Apply fine-grained column/row security | **Athena + Lake Formation** (see demo) |

---

*Next: [Instructor Demo — Lake Formation](demo-lake-formation.md)*  
*Day 20: [Kinesis Streaming Lab](../day20/lab-kinesis.md)*
