# Day 21 Lab: Amazon EMR — PySpark Analytics

**Type:** Attendee Lab  
**Duration:** ~60 minutes  
**Day:** 21 — Big Data Processing with EMR  
**Your Prefix:** `traineeNN` (replace `NN` with your number)

> **Before starting:** The instructor will provision your EMR cluster and share the cluster ID.  
> Write it here: `CLUSTER_ID = j-_______________`

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
    "Arn": "arn:aws:iam::<ACCOUNT_ID>:user/traineeNN"
}
```

> Replace `traineeNN` with your actual attendee ID throughout this lab.

---

## What You Will Build

By the end of this lab you will:

1. Inspect the EMR cluster the instructor pre-provisioned for you
2. Download and read the pre-written PySpark analytics job
3. Submit it as an EMR Step with custom S3 arguments
4. Monitor the step execution via CLI
5. Verify the Parquet analytics output in S3

---

## Architecture Overview

```
Input (from Day 19)                 EMR Cluster                    Output
traineeNN-2026-03-transformed    ─────────────────────────      traineeNN-2026-03-logs
  taxi_parquet/                     Spark 3.4 / Python 3.10           taxi_analytics/
    pickup_year=2024/            ──► emr_pyspark_job.py                 hourly/
    pickup_month=01/                   ▼                                   pickup_year=.../
      *.parquet                   Aggregation 1: hourly volumes            pickup_month=.../
                                  Aggregation 2: payment types   ──────►     *.parquet
                                       ▼                                 payment_type/
                             writes → S3 Parquet                           *.parquet

                             Logs → traineeNN-2026-03-logs/emr/
```

---

## Prerequisites

- AWS CLI configured
- Day 19 Glue ETL job completed — Parquet data is in `traineeNN-2026-03-transformed/`
- Instructor has shared your cluster ID (`j-XXXXXXXXXXXX`)

```bash
# Set your cluster ID as an environment variable (replace with real value)
export CLUSTER_ID=j-XXXXXXXXXXXX
export PREFIX=traineeNN
export BATCH=2026-03
```

---

## Part 1: Inspect the EMR Cluster (10 min)

### View cluster status

```bash
aws emr describe-cluster \
  --cluster-id $CLUSTER_ID \
  --query 'Cluster.{Name:Name,State:Status.State,StateMessage:Status.StateChangeReason.Message,Master:MasterPublicDnsName,LogUri:LogUri}'
```

Expected state: `WAITING` (cluster is running and idle, waiting for steps).

```bash
# View compute details
aws emr describe-cluster \
  --cluster-id $CLUSTER_ID \
  --query 'Cluster.{Release:ReleaseLabel,Applications:Applications[*].Name,MasterInstanceType:MasterInstanceType,CoreInstanceType:InstanceGroups[?InstanceGroupType==`CORE`].InstanceType}'
```

Expected:
- `ReleaseLabel`: `emr-6.15.0`
- `Applications`: `["Spark", "Hadoop"]`
- `MasterInstanceType`: `m5.xlarge`

```bash
# View security groups
aws emr describe-cluster \
  --cluster-id $CLUSTER_ID \
  --query 'Cluster.Ec2InstanceAttributes.{MasterSG:EmrManagedMasterSecurityGroup,SlaveSG:EmrManagedSlaveSecurityGroup,Profile:IamInstanceProfile}'
```

### View in console

1. Open **AWS Console** → **Amazon EMR** → **Clusters**
2. Click your cluster (the shared lab cluster is named `aws-de-lab-2026-03-emr-spark` — the instructor will also share this as `$CLUSTER_ID`)
3. Explore tabs:
   - **Summary:** Release, Applications, Log URI
   - **Hardware:** instance groups (master + core)
   - **Steps:** currently empty (you'll add one in Part 3)
   - **Application UIs:** links to Spark UI, YARN ResourceManager (if tunnel is set up)

---

## Part 2: Download and Inspect the PySpark Script (10 min)

### Download the script

```bash
aws s3 cp \
  s3://${PREFIX}-${BATCH}-scripts/emr/emr_pyspark_job.py \
  ~/emr_pyspark_job.py

cat ~/emr_pyspark_job.py
```

### Understand the script

The script performs **two aggregations** on the Parquet taxi data from Day 19:

**Aggregation 1 — Hourly trip volumes:**
```
Input: taxi_parquet/ (row per trip)
Output: taxi_analytics/hourly/
  pickup_year, pickup_month, pickup_hour,
  trip_count, avg_distance_miles, avg_fare_usd, total_fare_usd
```

**Aggregation 2 — Payment type breakdown:**
```
Input: taxi_parquet/ (row per trip)
Output: taxi_analytics/payment_type/
  pickup_year, pickup_month, payment_type,
  trip_count, avg_fare_usd
```

Both outputs are:
- Parquet format
- Partitioned by `pickup_year` / `pickup_month`
- Written with `overwrite` mode (re-runnable safely)

### Script arguments

| Argument | Value (your lab) |
|---|---|
| `--source-bucket` | `$PREFIX-$BATCH-transformed` |
| `--target-bucket` | `$PREFIX-$BATCH-logs` |
| `--database-name` | `$PREFIX_$BATCH_catalog` (e.g. `trainee01_2026_03_catalog`) |

> Note: The script writes analytics output to the `logs` bucket (`taxi_analytics/` prefix), not the `transformed` bucket. This separates raw, transformed, and analytics tiers.

---

## Part 3: Submit an EMR Step (15 min)

EMR Steps are units of work submitted to the cluster. Each step runs a Spark application in sequence (or parallel if you configure it).

### Submit the PySpark job as a step

```bash
STEP_ID=$(aws emr add-steps \
  --cluster-id $CLUSTER_ID \
  --steps '[{
    "Type": "Spark",
    "Name": "Taxi PySpark Analytics",
    "ActionOnFailure": "CONTINUE",
    "Args": [
      "--deploy-mode", "cluster",
      "--master", "yarn",
      "s3://'"${PREFIX}-${BATCH}"'-scripts/emr/emr_pyspark_job.py",
      "--source-bucket", "'"${PREFIX}-${BATCH}"'-transformed",
      "--target-bucket",  "'"${PREFIX}-${BATCH}"'-logs",
      "--database-name",  "'"${PREFIX//-/_}_${BATCH//-/_}"'_catalog"
    ]
  }]' \
  --query 'StepIds[0]' --output text)

echo "Step submitted: $STEP_ID"
```

> **`ActionOnFailure: CONTINUE`:** If this step fails, the cluster stays running (does not terminate). Other values: `TERMINATE_CLUSTER` (kills the cluster on failure — risky in shared labs), `CANCEL_AND_WAIT`.

### Monitor step execution

```bash
# Poll until the step reaches a terminal state
while true; do
  STATE=$(aws emr describe-step \
    --cluster-id $CLUSTER_ID \
    --step-id $STEP_ID \
    --query 'Step.Status.State' --output text)

  echo "$(date '+%H:%M:%S') Step state: $STATE"

  case "$STATE" in
    COMPLETED|FAILED|CANCELLED)
      break
      ;;
  esac
  sleep 20
done
```

Typical timeline:
- `PENDING` → `RUNNING`: 1–3 minutes (Spark driver starts, stage DAG built)
- `RUNNING`: 1–5 minutes (aggregation executes)
- `COMPLETED`: step done

### If the step fails

```bash
# View the step failure reason
aws emr describe-step \
  --cluster-id $CLUSTER_ID \
  --step-id $STEP_ID \
  --query 'Step.Status.FailureDetails'

# Get the log location
LOG_URI=$(aws emr describe-cluster \
  --cluster-id $CLUSTER_ID \
  --query 'Cluster.LogUri' --output text)

echo "Step logs are at: ${LOG_URI}steps/${STEP_ID}/"
aws s3 ls "${LOG_URI}steps/${STEP_ID}/" || echo "Logs may still be uploading..."
```

---

## Part 4: Check the Output (10 min)

### Verify Parquet output files exist in S3

```bash
# Hourly aggregation output
echo "=== Hourly partition structure ==="
aws s3 ls s3://${PREFIX}-${BATCH}-logs/taxi_analytics/hourly/ --recursive | head -20

# Payment type output
echo "=== Payment type partition structure ==="
aws s3 ls s3://${PREFIX}-${BATCH}-logs/taxi_analytics/payment_type/ --recursive | head -10
```

Expected structure:
```
taxi_analytics/hourly/pickup_year=2024/pickup_month=1/part-00000-xxxx.parquet
taxi_analytics/payment_type/pickup_year=2024/pickup_month=1/part-00000-xxxx.parquet
```

### Verify using Athena

In the Athena console (workgroup `$PREFIX-$BATCH-wg`):

> **Athena SQL note:** Replace `traineeNN_2026_03` with your actual catalog prefix (e.g. `trainee05_2026_03`) and `traineeNN-2026-03` with your bucket prefix before running.

```sql
-- First: create an external table pointing at the hourly analytics output
-- (the Glue crawler would normally do this automatically)
-- Replace traineeNN_2026_03 with your prefix, e.g. trainee05_2026_03
CREATE EXTERNAL TABLE IF NOT EXISTS
  "traineeNN_2026_03_catalog"."taxi_hourly_analytics"
(
  pickup_hour       INT,
  trip_count        BIGINT,
  avg_distance_miles DOUBLE,
  avg_fare_usd      DOUBLE,
  total_fare_usd    DOUBLE
)
PARTITIONED BY (pickup_year STRING, pickup_month STRING)
STORED AS PARQUET
LOCATION 's3://traineeNN-2026-03-logs/taxi_analytics/hourly/'
TBLPROPERTIES ('parquet.compress'='SNAPPY');
```

```sql
-- Load partitions
MSCK REPAIR TABLE "traineeNN_2026_03_catalog"."taxi_hourly_analytics";
```

```sql
-- Query the results
SELECT
    pickup_hour,
    trip_count,
    ROUND(avg_fare_usd, 2) AS avg_fare
FROM "traineeNN_2026_03_catalog"."taxi_hourly_analytics"
ORDER BY trip_count DESC
LIMIT 10;
```

---

## Part 5: Examine EMR Step Logs (10 min)

### View Spark driver logs from S3

EMR automatically pushes logs to S3 after step completion.

```bash
LOG_URI=$(aws emr describe-cluster \
  --cluster-id $CLUSTER_ID \
  --query 'Cluster.LogUri' --output text)

# List log files for your step
aws s3 ls "${LOG_URI}steps/${STEP_ID}/"

# Download and view the stderr log (Spark logs go to stderr by default)
aws s3 cp "${LOG_URI}steps/${STEP_ID}/stderr.gz" ~/step_stderr.gz
gunzip -c ~/step_stderr.gz | tail -50
```

Look for in the logs:
- `Reading Parquet from s3://...` — confirms input path
- `Writing hourly aggregations to ...` — confirms first write started
- `Wrote N hourly rows and M payment-type rows.` — confirms completion
- `PySpark job complete.` — final line of the script

### Check Spark application history

```bash
# List your steps and their execution times
aws emr list-steps \
  --cluster-id $CLUSTER_ID \
  --query 'Steps[*].{Name:Name,State:Status.State,StartTime:Status.Timeline.StartDateTime,EndTime:Status.Timeline.EndDateTime}' \
  --output table
```

---

## Part 6: Cancel a Running Step (optional exercise)

Attendees can cancel their own steps.

```bash
# Submit a long-running step (sleep 300 = 5 minutes)
LONG_STEP_ID=$(aws emr add-steps \
  --cluster-id $CLUSTER_ID \
  --steps '[{
    "Type": "CUSTOM_JAR",
    "Name": "Long running test step",
    "ActionOnFailure": "CONTINUE",
    "Jar": "command-runner.jar",
    "Args": ["sleep", "300"]
  }]' \
  --query 'StepIds[0]' --output text)

echo "Long step submitted: $LONG_STEP_ID"

# Verify it's running
aws emr describe-step \
  --cluster-id $CLUSTER_ID \
  --step-id $LONG_STEP_ID \
  --query 'Step.Status.State'

# Cancel it
aws emr cancel-steps \
  --cluster-id $CLUSTER_ID \
  --step-ids $LONG_STEP_ID

# Confirm cancelled
aws emr describe-step \
  --cluster-id $CLUSTER_ID \
  --step-id $LONG_STEP_ID \
  --query 'Step.Status.State'
```

---

## Validation Checklist

```bash
# 1. PySpark step completed
aws emr describe-step \
  --cluster-id $CLUSTER_ID \
  --step-id $STEP_ID \
  --query 'Step.Status.State'
# Expected: COMPLETED

# 2. Analytics output files exist
aws s3 ls s3://${PREFIX}-${BATCH}-logs/taxi_analytics/ --recursive | wc -l
# Expected: > 0

# 3. Hourly and payment_type directories both exist
aws s3 ls s3://${PREFIX}-${BATCH}-logs/taxi_analytics/
```

---

## EMR vs Glue — When to Use Each

| Scenario | EMR | Glue |
|---|---|---|
| Large Spark job with custom libraries | ✅ (Docker or bootstrap) | Partial (limited to supported packages) |
| Pure ETL with AWS-managed infrastructure | Possible | ✅ Simpler |
| Multi-framework (Spark + HBase + Flink + Presto) | ✅ | ❌ Spark only |
| Cost at scale (> 10 TiB/day) | ✅ Spot instances | ❌ DPU pricing becomes expensive |
| No-ops, schema inference, Glue catalog integration | Partial | ✅ Native |
| Custom Spark configurations and tuning | ✅ Full control | Limited |
| Training ML models (Spark MLlib, SageMaker) | ✅ | ❌ |

---

*Next: [Instructor Demo — EMR Advanced](demo-emr-advanced.md)*
