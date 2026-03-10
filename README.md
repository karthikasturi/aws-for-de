# AWS Data Engineering — Lab and Demo Guides

Training guides for the 7-day AWS Data Engineering course.

**Account:** `<ACCOUNT_ID>` | **Region:** `ap-south-1` (Mumbai) | **Batch:** `<BATCH_ID>`

---

## Guide Types

| Badge | Meaning |
|---|---|
| 🧪 **Lab** | Attendee self-service — you run this yourself |
| 🎬 **Demo** | Instructor-led — watch and take notes |

---

## Schedule

### Day 14 — AWS 10,000 Feet Overview

| Guide | Type | Duration |
|---|---|---|
| [Console Exploration](labs/day14/lab-console-exploration.md) | 🧪 Lab | 45 min |
| [Billing and Cost Explorer](labs/day14/demo-billing-cost-explorer.md) | 🎬 Demo | 15 min |

**Topics covered:**
- AWS global infrastructure: Regions, AZs, Edge Locations
- AWS service categories: Compute, Storage, Networking, Analytics
- Key DE services: S3, Glue, Athena, EMR, Redshift, Kinesis, DMS, MWAA
- IAM basics: your user, your policy, your roles

---

### Day 15 — Identity and Access Management (IAM)

| Guide | Type | Duration |
|---|---|---|
| [IAM Role Architecture and Service Roles](labs/day15/demo-iam-role-creation.md) | 🎬 Demo | 60 min |
| [Explore and Verify IAM Roles](labs/day15/lab-iam-explore-and-verify.md) | 🧪 Lab | 45 min |
| [Billing Alerts and Budget Management](labs/day15/demo-billing-budget-alerts.md) | 🎬 Demo | 20 min |

**Topics covered:**
- IAM Users, Groups, Roles, Policies
- Trust policies vs permission policies
- Least privilege and resource-level scoping
- `iam:PassRole` — why it's required
- EMR two-role model (service role + EC2 instance profile)
- Policy evaluation: Allow vs explicit Deny vs implicit Deny
- Billing alerts and AWS Budgets
- Cost Anomaly Detection

---

### Day 16 — AWS Object Storage (S3, Glacier, and CloudFront)

| Guide | Type | Duration |
|---|---|---|
| [S3 Core Operations](labs/day16/lab-s3-core-operations.md) | 🧪 Lab | 90 min |
| [S3 Advanced Features](labs/day16/demo-s3-advanced-features.md) | 🎬 Demo | 45 min |

**Topics covered:**
- S3 buckets, objects, storage classes, and naming conventions
- Versioning, delete markers, and version restore
- Lifecycle rules — transitions to Glacier, automatic expiration
- SSE-KMS encryption and Bucket Keys
- Server access logging and CloudTrail data events (demo)
- Cross-attendee access control and presigned URLs
- CloudFront + S3 Origin Access Control (demo)
- S3 Cross-Region Replication (demo)
- Snowball and Transfer Acceleration (demo)

---

## Access Model

All attendees share one AWS sandbox account. Each attendee's resources are namespaced by their prefix (e.g. `trainee01`).

| What attendees CAN do | What attendees CANNOT do |
|---|---|
| Navigate all service consoles | Create/modify IAM roles or policies |
| Read/write their own S3 buckets (`traineeNN-*`) | Access another attendee's resources |
| Run CLI commands against their own resources | View billing or Cost Explorer |
| Inspect IAM roles and policies (read-only) | Create IAM users, groups, or policies |
| Submit EMR steps to their cluster | Create budget alerts |

> These constraints are enforced by the per-attendee IAM policy managed by the instructor (scoped to each attendee's prefix).

---

## Instructor Setup — Before Each Day

```bash
export AWS_PROFILE=aws-de-lab
aws sts get-caller-identity   # verify Account: "<ACCOUNT_ID>"
```

| Before | Action |
|---|---|
| **Day 14** | Pre-provision EC2 workstations (one per attendee) via IAM Console + EC2 Console |
| **Day 15** | Pre-provision 6 IAM service roles per attendee via IAM Console |
| **Day 16** | Pre-provision 7 S3 buckets + KMS key per attendee via S3 Console (stay up for entire course) |
| **Day 17** | Pre-provision PostgreSQL RDS instance per attendee via RDS Console; update IAM Glue/EMR role policies with real S3 ARNs |
| **Day 18** | Pre-provision Redshift cluster per attendee via Redshift Console |
| **Day 19** | Pre-provision Glue catalog, Athena workgroup, Lake Formation settings |
| **Day 20** | Pre-provision Kinesis stream + Data Firehose delivery stream |
| **Day 21** | Pre-provision EMR cluster per attendee via EMR Console |

**After each day:** Tear down day-specific resources from the console to avoid overnight charges. S3 buckets and IAM roles stay up for the entire course.

---

---

### Day 17 — Amazon RDS — PostgreSQL

| Guide | Type | Duration |
|---|---|---|
| [RDS PostgreSQL](labs/day17/lab-rds-postgresql.md) | 🧪 Lab | 90 min |
| [RDS Advanced — Multi-AZ, Read Replicas, Aurora](labs/day17/demo-rds-advanced.md) | 🎬 Demo | 60 min |

**Topics covered:**
- Retrieve RDS credentials from AWS Secrets Manager
- Connect with `psql` using environment variables
- Create tables, insert data, run analytical queries
- Index creation, `EXPLAIN ANALYZE`, views, JOINs
- Parameter group verification (slow query logging)
- Manual snapshots and CloudWatch metrics
- Demo: Multi-AZ deployment and failover sequence
- Demo: Read Replicas — async replication, cross-region
- Demo: Performance Insights — DB Load, Top SQL, Top waits
- Demo: PITR (Point-in-Time Recovery)
- Demo: RDS Proxy — connection pooling for Lambda/serverless
- Demo: RDS vs Aurora feature comparison

---

### Day 18 — Amazon Redshift — Data Warehousing

| Guide | Type | Duration |
|---|---|---|
| [Redshift Data Warehousing](labs/day18/lab-redshift.md) | 🧪 Lab | 90 min |
| [Redshift Advanced — WLM, Concurrency Scaling, Serverless](labs/day18/demo-redshift-advanced.md) | 🎬 Demo | 45 min |

**Topics covered:**
- Retrieve Redshift credentials from Secrets Manager
- Connect via Query Editor v2 and `psql` (port 5439)
- Design fact tables with `DISTKEY` and `SORTKEY`
- Column encoding (ZSTD, BYTEDICT) for compression
- `COPY` command — bulk load from S3 with IAM role
- Analytical queries and `EXPLAIN` plan — DS_DIST_NONE vs DS_BCAST_INNER
- `VACUUM` and `ANALYZE` for cluster maintenance
- Redshift Spectrum — query S3 via the Glue Data Catalog
- Demo: WLM queues — Automatic WLM vs manual configuration
- Demo: Concurrency Scaling — burst capacity at no extra cost
- Demo: Redshift Serverless — RPUs, scale-to-zero, vs provisioned
- Demo: Elastic Resize vs Classic Resize
- Demo: Data Sharing — cross-cluster live data access

---

### Day 19 — AWS Glue + Athena + Lake Formation

| Guide | Type | Duration |
|---|---|---|
| [Glue + Athena — Serverless ETL and Query](labs/day19/lab-glue-athena.md) | 🧪 Lab | 90 min |
| [Lake Formation — Fine-Grained Access Control](labs/day19/demo-lake-formation.md) | 🎬 Demo | 45 min |

**Topics covered:**
- Explore Glue catalog database, crawler, and ETL job configuration
- Read and understand the pre-written PySpark ETL script
- Run the Glue crawler to auto-discover CSV schema in S3
- Query raw CSV data with Athena (workgroup + scan limits)
- Run the Glue ETL job: CSV → Parquet with partition columns
- Verify transformed Parquet in S3 and the Glue catalog table
- Query Parquet with Athena — data scanned vs CSV comparison
- Demo: Lake Formation architecture — registered S3 locations, admins
- Demo: Column-level security — `no_location_data` filter (hide GPS coords)
- Demo: Row-level security — payment type isolation
- Demo: LF-Tags — attribute-based access control at scale
- Demo: CloudTrail audit trail for data access events

---

### Day 20 — Amazon Kinesis — Real-Time Streaming

| Guide | Type | Duration |
|---|---|---|
| [Kinesis Data Streams + Firehose](labs/day20/lab-kinesis.md) | 🧪 Lab | 60 min |
| [AWS Batch — Managed Batch Computing](labs/day20/demo-batch.md) | 🎬 Demo | 30 min |

**Topics covered:**
- Inspect Kinesis data stream (1 shard PROVISIONED, KMS, 24h retention)
- Understand shard throughput limits (1 MB/s write, 2 MB/s read)
- Run the `kinesis_producer.py` script to send synthetic taxi events
- Monitor IncomingRecords and IncomingBytes via CloudWatch
- Read raw records via `GetShardIterator` + `GetRecords` (Base64 decode)
- Wait for Firehose 60-second buffer flush → Parquet files in S3
- Verify delivery metrics and query Firehose output with Athena
- Demo: AWS Batch architecture — job queues, compute environments, job definitions
- Demo: Managed vs Fargate compute environments
- Demo: Spot instances with `SWITCH_TO_ON_DEMAND` fallback
- Demo: Array jobs — parallel processing across partitions
- Demo: Batch vs Lambda vs EMR vs Glue comparison

---

### Day 21 — Amazon EMR — Big Data with PySpark

| Guide | Type | Duration |
|---|---|---|
| [EMR PySpark Analytics](labs/day21/lab-emr-pyspark.md) | 🧪 Lab | 60 min |
| [EMR Advanced — Cluster Creation, EMR Studio, Auto-termination](labs/day21/demo-emr-advanced.md) | 🎬 Demo | 45 min |

**Topics covered:**
- Inspect the pre-provisioned EMR cluster (emr-6.15.0, Spark 3.4)
- Download and read the PySpark analytics script from S3
- Submit an EMR Step: PySpark job with `--source-bucket` / `--target-bucket` args
- Monitor step execution with `describe-step` and CloudWatch
- View Spark step logs from S3 (stderr.gz)
- Cancel a running step — attendee-level capability
- Verify hourly and payment-type Parquet analytics in S3
- Create Athena external table over EMR output, run queries
- Demo: EMR cluster creation walkthrough — node types, bootstrap, auto-termination
- Demo: Instance Fleets with Spot and `SWITCH_TO_ON_DEMAND` fallback
- Demo: EMR Studio — JupyterHub + PySpark kernel attached to cluster
- Demo: EMR Serverless — scan-to-zero, RPU capacity, vs provisioned
- Demo: Cost model — Spot strategy, idle timeout, S3-first design

---

## Full Guide Index

| Day | Lab | Demo |
|---|---|---|
| 14 | [Console Exploration](labs/day14/lab-console-exploration.md) | [Billing and Cost Explorer](labs/day14/demo-billing-cost-explorer.md) |
| 15 | [IAM Explore and Verify](labs/day15/lab-iam-explore-and-verify.md) | [IAM Role Architecture](labs/day15/demo-iam-role-creation.md) · [Billing Alerts](labs/day15/demo-billing-budget-alerts.md) |
| 16 | [S3 Core Operations](labs/day16/lab-s3-core-operations.md) | [S3 Advanced Features](labs/day16/demo-s3-advanced-features.md) |
| 17 | [RDS PostgreSQL](labs/day17/lab-rds-postgresql.md) | [RDS Advanced](labs/day17/demo-rds-advanced.md) |
| 18 | [Redshift Data Warehousing](labs/day18/lab-redshift.md) | [Redshift Advanced](labs/day18/demo-redshift-advanced.md) |
| 19 | [Glue + Athena](labs/day19/lab-glue-athena.md) | [Lake Formation](labs/day19/demo-lake-formation.md) |
| 20 | [Kinesis Streaming](labs/day20/lab-kinesis.md) | [AWS Batch](labs/day20/demo-batch.md) |
| 21 | [EMR PySpark](labs/day21/lab-emr-pyspark.md) | [EMR Advanced](labs/day21/demo-emr-advanced.md) |
