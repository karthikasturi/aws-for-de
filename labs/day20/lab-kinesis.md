# Day 20 Lab: Amazon Kinesis — Real-Time Data Streaming

**Type:** Attendee Lab  
**Duration:** ~60 minutes  
**Day:** 20 — Real-Time Data Streaming  
**Your Prefix:** `traineeNN` (replace `NN` with your number)

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

1. Explore your pre-provisioned Kinesis data stream and Firehose delivery stream
2. Understand shard-based partitioning and throughput limits
3. Use the pre-written producer script to send synthetic taxi trip events
4. Monitor stream throughput using CloudWatch metrics
5. Verify that Firehose has delivered Parquet files to S3

---

## Architecture Overview

```
Producer Script                  Kinesis            Firehose              S3
(kinesis_producer.py)            Data Stream        Delivery Stream       Parquet Landing
                                 1 shard                                  ┌───────────────────┐
 generate_event()                PROVISIONED        60-second buffer      │ firehose-landing/ │
  vendor_id, pickup_datetime ───► put_record()  ───► convert to ─────────► taxi/             │
  trip_distance, fare_amount     ▲                   Parquet via           │   yyyy/mm/dd/hh/ │
                                 |                   Glue schema           │     *.parquet    │
 (50 req/sec per shard)          |                                         └───────────────────┘
 (1 MB/sec write limit)         consumer apps
                                 could also read here
                                 (Flink, Lambda, KCL)
```

---

## Prerequisites

- Python 3 installed (`python3 --version`)
- `boto3` installed (`pip3 install boto3`)
- AWS CLI configured (`aws configure list`)

---

## Part 1: Explore Your Kinesis Infrastructure (10 min)

### Inspect the data stream

```bash
# View stream details
aws kinesis describe-stream-summary \
  --stream-name traineeNN-2026-03-stream \
  --query 'StreamDescriptionSummary.{Status:StreamStatus,ShardCount:OpenShardCount,RetentionHours:RetentionPeriodHours,Encryption:EncryptionType}'
```

Expected output:
```json
{
    "Status": "ACTIVE",
    "ShardCount": 1,
    "RetentionHours": 24,
    "Encryption": "KMS"
}
```

```bash
# View shard details (partition key range)
aws kinesis list-shards \
  --stream-name traineeNN-2026-03-stream \
  --query 'Shards[*].{ShardId:ShardId,StartHash:HashKeyRange.StartingHashKey,EndHash:HashKeyRange.EndingHashKey}'
```

> **Shard throughput limits:**
> - Write: 1,000 records/sec OR 1 MB/sec per shard (whichever is reached first)
> - Read: 5 transactions/sec OR 2 MB/sec per shard (shared across all consumers)

### Inspect the Firehose delivery stream

```bash
aws firehose describe-delivery-stream \
  --delivery-stream-name traineeNN-2026-03-firehose \
  --query 'DeliveryStreamDescription.{Status:DeliveryStreamStatus,Source:Source,Destination:Destinations[0].ExtendedS3DestinationDescription.{Bucket:BucketARN,Prefix:Prefix,BufferingHints:BufferingHints,DataFormatConversionConfiguration:DataFormatConversionConfiguration.Enabled}}'
```

Key settings to note:
- **BufferingHints.IntervalInSeconds: 60** — Firehose waits 60 seconds (or 64 MB, whichever comes first) before writing to S3
- **DataFormatConversionConfiguration: enabled** — converts JSON → Parquet using the Glue schema
- **BucketARN:** points to `traineeNN-2026-03-firehose-landing`

```bash
# Check the Glue table Firehose uses for schema reference
aws firehose describe-delivery-stream \
  --delivery-stream-name traineeNN-2026-03-firehose \
  --query 'DeliveryStreamDescription.Destinations[0].ExtendedS3DestinationDescription.DataFormatConversionConfiguration.SchemaConfiguration'
```

---

## Part 2: Download and Run the Producer Script (15 min)

### Download the producer

```bash
# Download the producer script from S3
aws s3 cp \
  s3://traineeNN-2026-03-scripts/kinesis/kinesis_producer.py \
  ~/kinesis_producer.py

# Inspect it
cat ~/kinesis_producer.py
```

The script generates synthetic events with these fields:
- `vendor_id` — "CMT" or "VTS"
- `pickup_datetime` — current UTC time
- `passenger_count` — 1 to 4
- `trip_distance` — 0.5 to 15.0 miles
- `payment_type` — "Credit" or "Cash"
- `fare_amount` — $5 to $50

### Run the producer — small test batch

```bash
python3 ~/kinesis_producer.py \
  --stream traineeNN-2026-03-stream \
  --count 20 \
  --region ap-south-1 \
  --delay 0.1
```

You should see output like:
```
Sending 20 events to stream: traineeNN-2026-03-stream
  [   1/20] 2024-01-15T10:23:45.123456+00:00  fare=$23.40
  [   2/20] 2024-01-15T10:23:45.234567+00:00  fare=$12.80
  ...
```

### Send a larger batch

```bash
# Send 200 events with 0.5 second spacing (~100 seconds total)
python3 ~/kinesis_producer.py \
  --stream traineeNN-2026-03-stream \
  --count 200 \
  --region ap-south-1 \
  --delay 0.5
```

Leave this running — continue with Part 3 in a **second terminal window**.

---

## Part 3: Monitor the Stream in Real Time (10 min)

### Check CloudWatch metrics via CLI

While the producer is running (or just after), check the stream metrics:

```bash
# Incoming records in the last 5 minutes
aws cloudwatch get-metric-statistics \
  --namespace AWS/Kinesis \
  --metric-name IncomingRecords \
  --dimensions Name=StreamName,Value=traineeNN-2026-03-stream \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Sum \
  --query 'Datapoints[*].{Time:Timestamp,Records:Sum}' \
  --output table
```

```bash
# Incoming bytes in the last 5 minutes
aws cloudwatch get-metric-statistics \
  --namespace AWS/Kinesis \
  --metric-name IncomingBytes \
  --dimensions Name=StreamName,Value=traineeNN-2026-03-stream \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Sum \
  --query 'Datapoints[*].{Time:Timestamp,Bytes:Sum}' \
  --output table
```

### Monitor in console

1. Open **AWS Console** → **Amazon Kinesis** → **Data streams**
2. Click `traineeNN-2026-03-stream`
3. **Monitoring** tab → observe:
   - **Put records — success**: real-time count of records written
   - **Incoming data — bytes**: bytes per second
   - **Iterator age — milliseconds**: lag of the oldest consumer; should be 0 if no consumers

---

## Part 4: Read a Record Directly from the Stream (10 min)

You can manually read records from the stream using the Kinesis GetShardIterator + GetRecords API pair.

```bash
# Step 1: Get the shard ID
SHARD_ID=$(aws kinesis list-shards \
  --stream-name traineeNN-2026-03-stream \
  --query 'Shards[0].ShardId' --output text)
echo "Shard: $SHARD_ID"

# Step 2: Get a shard iterator (TRIM_HORIZON = from oldest record in retention window)
ITERATOR=$(aws kinesis get-shard-iterator \
  --stream-name traineeNN-2026-03-stream \
  --shard-id $SHARD_ID \
  --shard-iterator-type TRIM_HORIZON \
  --query 'ShardIterator' --output text)

# Step 3: Read up to 10 records
aws kinesis get-records \
  --shard-iterator $ITERATOR \
  --limit 10 \
  --query 'Records[*].Data' --output text \
  | while read line; do echo "$line" | base64 -d; echo; done
```

Expected output (one JSON per line):
```json
{"vendor_id": "CMT", "pickup_datetime": "2024-01-15T10:23:45.123456+00:00", "passenger_count": 2, "trip_distance": 7.3, "payment_type": "Credit", "fare_amount": 28.50}
```

> **Note:** Raw records are Base64-encoded in the Kinesis API response. The `-d` decode gives you the original JSON bytes.

### Shard iterator types explained

| Type | What it returns |
|---|---|
| `TRIM_HORIZON` | From the oldest retained record (~24 hours back) |
| `LATEST` | Only records written _after_ you call GetShardIterator |
| `AT_SEQUENCE_NUMBER` | From a specific sequence number |
| `AFTER_SEQUENCE_NUMBER` | After a specific sequence number |
| `AT_TIMESTAMP` | From a specific timestamp (useful for replay) |

---

## Part 5: Check Firehose Delivery to S3 (10 min)

Firehose buffers records for 60 seconds before writing to S3. After your producer run, wait at least 90 seconds, then check.

### Wait for the buffer flush

```bash
echo "Waiting 90 seconds for Firehose to flush buffer..."
sleep 90
```

### Check for Parquet files

```bash
# List all files in the Firehose landing bucket
aws s3 ls s3://traineeNN-2026-03-firehose-landing/taxi/ --recursive

# Expected structure (Hive-partitioned):
# taxi/year=YYYY/month=MM/day=DD/traineeNN-2026-03-firehose-1-YYYY-MM-DD-HH-MM-SS-xxxx.parquet
```

```bash
# Count files
aws s3 ls s3://traineeNN-2026-03-firehose-landing/taxi/ --recursive | wc -l
```

### Check Firehose delivery metrics

```bash
# Records successfully delivered to S3
aws cloudwatch get-metric-statistics \
  --namespace AWS/Firehose \
  --metric-name DeliveryToS3.Records \
  --dimensions Name=DeliveryStreamName,Value=traineeNN-2026-03-firehose \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Sum \
  --query 'sort_by(Datapoints, &Timestamp)[*].{Time:Timestamp,Delivered:Sum}' \
  --output table
```

```bash
# Failed delivery attempts (should be 0)
aws cloudwatch get-metric-statistics \
  --namespace AWS/Firehose \
  --metric-name DeliveryToS3.DataFreshness \
  --dimensions Name=DeliveryStreamName,Value=traineeNN-2026-03-firehose \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Average \
  --query 'Datapoints[*]' \
  --output table
```

---

## Part 6: Query Streamed Data with Athena (5 min)

If the Glue catalog is correctly set up (from Day 19), Athena can query the Parquet files that Firehose wrote.

```sql
-- In Athena console with workgroup traineeNN-2026-03-wg
-- Query the Firehose-delivered records (if the Glue table was created)

SELECT
    vendor_id,
    payment_type,
    COUNT(*)                     AS trip_count,
    SUM(CAST(fare_amount AS DOUBLE)) AS total_fare
FROM "traineeNN_2026_03_catalog"."firehose_taxi"
GROUP BY vendor_id, payment_type
ORDER BY vendor_id;
```

> **If the table doesn't exist yet:** A dedicated Firehose crawler was provisioned on Day 20.  
> Run it after your producer has pushed some data and Firehose has flushed (allow ~90 seconds):
> ```bash
> # Start the Firehose landing crawler
> aws glue start-crawler --name traineeNN-2026-03-firehose-crawler
>
> # Poll until READY (takes ~1-2 minutes)
> aws glue get-crawler \
>   --name traineeNN-2026-03-firehose-crawler \
>   --query 'Crawler.{State:State,LastStatus:LastCrawl.Status}'
> ```
> Once the crawler shows `"State": "READY"` and `"LastStatus": "SUCCEEDED"`, refresh Athena and the `firehose_taxi` table will appear.

---

## Validation Checklist

```bash
# 1. Stream is ACTIVE
aws kinesis describe-stream-summary \
  --stream-name traineeNN-2026-03-stream \
  --query 'StreamDescriptionSummary.StreamStatus'

# 2. Records were sent (check last 15 minutes)
aws cloudwatch get-metric-statistics \
  --namespace AWS/Kinesis \
  --metric-name IncomingRecords \
  --dimensions Name=StreamName,Value=traineeNN-2026-03-stream \
  --start-time $(date -u -d '15 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 900 \
  --statistics Sum \
  --query 'Datapoints[*].Sum'

# 3. Parquet files exist in S3
aws s3 ls s3://traineeNN-2026-03-firehose-landing/taxi/ --recursive | wc -l
```

---

## Kinesis Stream vs SQS vs SNS — Quick Comparison

| | Kinesis Data Streams | SQS | SNS |
|---|---|---|---|
| **Model** | Ordered log per shard | Queue (pull) | Pub/sub (push) |
| **Retention** | 24h – 365 days | Up to 14 days | No storage |
| **Consumers** | Multiple, independent position | One consumer per message | Multiple subscribers |
| **Ordering** | Per shard | Not guaranteed (FIFO queue optional) | No ordering |
| **Replay** | Yes (within retention) | No | No |
| **Throughput** | Fixed per shard (1MB/s write) | Virtually unlimited | Unlimited |
| **Best for** | Event streaming, analytics, ML | Task queues, decoupling | Notifications, fan-out |

---

*Next: [Instructor Demo — AWS Batch](demo-batch.md)*  
*Day 21: [EMR PySpark Lab](../day21/lab-emr-pyspark.md)*
