# Day 14 Lab: Explore the AWS Console

**Type:** Attendee Self-Service Lab  
**Duration:** ~45 minutes  
**Day:** 14 — AWS 10,000 Feet Overview  
**Prerequisites:** Received AWS credentials from the instructor

---

## Overview

In this lab you will log into the AWS Management Console for the first time, orient yourself to its layout, and locate the key data engineering services you will use throughout this course.

> **Shared Account Notice**  
> All attendees share a single AWS sandbox account (`<ACCOUNT_ID>`, region `ap-south-1`).  
> Every resource you create or see will be prefixed with your attendee ID (e.g. `trainee01`).  
> You will not be able to see or modify other attendees' resources — your IAM policy enforces this.

---

## Part 1: First Login

### 1.1 Sign in to the Console

Open your browser and navigate to:

```
https://<ACCOUNT_ID>.signin.aws.amazon.com/console
```

| Field | Value |
|---|---|
| **Account ID** | `<ACCOUNT_ID>` |
| **IAM Username** | Your attendee ID (e.g. `trainee01`) |
| **Password** | Provided by instructor — you will be prompted to change it on first login |

### 1.2 Set your new password

AWS will ask you to choose a new password on first login. Pick something strong (12+ characters, mixed case, numbers, symbols).

### 1.3 Set the region

In the top-right corner, click the region selector and choose:

**Asia Pacific (Mumbai) — `ap-south-1`**

> All lab resources are deployed in `ap-south-1`. Always confirm this before any console action.

---

## Part 2: Console Layout Orientation

Take 5 minutes to explore the console home page.

### What you will see

| Area | Description |
|---|---|
| **Top navigation bar** | Search, services menu, region selector, account info |
| **Console Home** | Recently visited services, AWS news, cost widgets |
| **Services menu** (top-left ☰) | Categorised list of all 200+ AWS services |

### Try the Search Bar

Click the **search bar** at the top (or press `Alt+S`) and search for each of these services — notice how fast you can jump directly to a service:

- `S3`
- `Glue`
- `Athena`
- `EMR`
- `Redshift`
- `Kinesis`
- `IAM`
- `CloudWatch`

---

## Part 3: Explore Key Data Engineering Services

For each service below, navigate to the console page and answer the checkpoint questions.

### 3.1 Amazon S3 — Object Storage

**Navigate:** Search → `S3`

**What to observe:**
- The S3 console has **no region selector** — it lists all your buckets across all AWS regions in one view. This is a UI design choice: S3 bucket names are globally unique, so AWS can show them all without needing a region filter.
- **Important:** S3 is a **regional service** — your data is stored in the specific region you chose when creating the bucket and stays there unless you explicitly replicate it. A bucket created in `ap-south-1` holds its data in Mumbai.
- At this point (Day 14), you will see no buckets with your prefix yet. S3 buckets are provisioned on **Day 16**.

**Checkpoint questions:**
1. If S3 shows all buckets without a region filter, does that make it a "global service" like IAM or Route 53? Why or why not?
2. What is the maximum size of a single S3 object? What upload method is required for objects above 5 GB?

---

### 3.2 AWS Glue — Serverless ETL and Data Catalog

**Navigate:** Search → `Glue` → AWS Glue Studio

**What to observe:**
- Left sidebar: Data Catalog, ETL Jobs, Crawlers, Connections
- Glue is a **regional service** — confirm `ap-south-1` in the top-right

**Checkpoint questions:**
1. What is the difference between a Glue **Crawler** and a Glue **Job**?
2. Where does Glue store table metadata? (Hint: look for "Data Catalog")

---

### 3.3 Amazon Athena — Serverless SQL Query

**Navigate:** Search → `Athena` → Query editor

**What to observe:**
- The query editor is where you run SQL against S3 data via the Glue Data Catalog
- Notice the **Workgroup** selector (top-right of editor). Yours is `traineeNN-<BATCH_ID>-workgroup` (provisioned on Day 19).

**Checkpoint questions:**
1. Athena is described as "serverless." What do you think this means in terms of infrastructure you manage?
2. What format does Athena charge by — rows scanned, time, or data scanned?

---

### 3.4 Amazon EMR — Managed Spark/Hadoop

**Navigate:** Search → `EMR` → Clusters

**What to observe:**
- EMR clusters are provisioned on **Day 21**
- Notice the **Cluster states**: Starting, Running, Waiting, Terminating, Terminated

**Checkpoint questions:**
1. What is the difference between a **Primary node** and a **Core node** in EMR?
2. What open-source frameworks does EMR support? (Hint: click **Create cluster** and look at the software configuration options — then cancel without creating)

---

### 3.5 Amazon Redshift — Cloud Data Warehouse

**Navigate:** Search → `Redshift` → Clusters

**What to observe:**
- Redshift clusters are provisioned on **Day 18**
- Notice **Redshift Serverless** vs **Provisioned** tabs

**Checkpoint questions:**
1. What is the difference between Redshift Provisioned and Redshift Serverless?
2. What is a **Redshift Spectrum** and how does it relate to S3?

---

### 3.6 Amazon Kinesis — Real-Time Data Streaming

**Navigate:** Search → `Kinesis`

**What to observe:**
- The Kinesis family consists of four services:
  - **Kinesis Data Streams** — real-time data ingestion with consumer control
  - **Amazon Data Firehose** — load streaming data to S3/Redshift/Splunk (formerly *Kinesis Data Firehose*, renamed Nov 2023)
  - **Amazon Managed Service for Apache Flink** — real-time stream processing (formerly *Kinesis Data Analytics*, renamed Aug 2023)
  - **Kinesis Video Streams** — streaming video from connected devices
- Kinesis Data Streams and Amazon Data Firehose are used in **Day 20**

**Checkpoint questions:**
1. What is the difference between Kinesis Data Streams and Amazon Data Firehose? When would you use each?
2. What is a **shard** in Kinesis Data Streams? What read/write throughput does one shard provide?

---

### 3.7 AWS DMS — Database Migration Service

**Navigate:** Search → `DMS` → Replication instances

**What to observe:**
- DMS orchestrates ongoing data replication between source and target databases
- Used to migrate from on-prem or RDS into Redshift/S3

**Checkpoint questions:**
1. What is a DMS **Replication Task**?
2. What is the difference between **Full load** and **CDC (Change Data Capture)** migration modes?

---

### 3.8 Amazon MWAA — Managed Airflow

**Navigate:** Search → `MWAA` (or "Managed Apache Airflow")

**What to observe:**
- MWAA provides Apache Airflow as a fully managed service — no need to run your own Airflow servers
- DAGs are stored in an S3 bucket

**Checkpoint questions:**
1. How does MWAA differ from managing your own Airflow on EC2?
2. Where are Airflow DAG files stored in MWAA?

---

## Part 4: IAM — Your User and Permissions

**Navigate:** Search → `IAM`

> **Note:** Your attendee policy grants read-only IAM access. You can explore roles and policies but cannot create or modify them.

### 4.1 Inspect your IAM user

1. In IAM, navigate to **Users** → click your username (e.g. `trainee01`)
2. Click the **Permissions** tab — you will see a policy named `trainee01-lab-policy`
3. Click the policy name → **JSON** tab to read the full policy document

**What to look for:**
- Notice the `Deny` statement at the bottom — this protects the shared account
- Notice how every `Allow` statement is scoped to your prefix (e.g. `trainee01-*`)

### 4.2 Inspect pre-created IAM roles

1. Navigate to **Roles**
2. Search for your prefix (e.g. `trainee01`) in the search box

After **Day 15**, the instructor will have set up these roles for you:

| Role Name | Purpose |
|---|---|
| `traineeNN-<BATCH_ID>-glue-service-role` | Glue crawlers and ETL jobs |
| `traineeNN-<BATCH_ID>-emr-service-role` | EMR cluster management |
| `traineeNN-<BATCH_ID>-emr-ec2-role` | EMR EC2 worker nodes |
| `traineeNN-<BATCH_ID>-redshift-s3-role` | Redshift COPY/Spectrum access |
| `traineeNN-<BATCH_ID>-lakeformation-role` | Lake Formation service |
| `traineeNN-<BATCH_ID>-firehose-role` | Amazon Data Firehose delivery |

> If you don't see these yet — the instructor hasn't provisioned them yet. This is covered in detail on **Day 15**.

---

## Part 5: CloudWatch — Logs and Monitoring

**Navigate:** Search → `CloudWatch`

**What to observe:**
- **Dashboards**: pre-built and custom visualisations
- **Log Groups**: where service logs land (Glue, EMR, RDS all write here)
- **Alarms**: threshold-based alerts (CPU, error rates, etc.)

**Checkpoint questions:**
1. Where would you look in CloudWatch to debug a failed Glue job?
2. What is the difference between a CloudWatch **Metric** and a CloudWatch **Log**?

---

## Part 6: VPC — Networking Basics

**Navigate:** Search → `VPC` → Your VPCs

**What to observe:**
- The lab uses a pre-existing VPC (`vpc-0804653879b35d77a`)
- **Private subnets**: where RDS, Redshift, EMR run (no direct internet)
- **Public subnets**: where workstation EC2 instances run

**Key concepts to note:**
- Subnets are in Availability Zones (AZs) — `ap-south-1a`, `ap-south-1b`
- Security Groups = stateful firewall rules per resource
- No NAT Gateway exists in this lab — cost control

---

## Wrap-Up Checklist

Before leaving this lab, confirm you can:

- [ ] Log into the AWS Console with your attendee credentials
- [ ] Switch to the `ap-south-1` region
- [ ] Navigate directly to S3, Glue, Athena, EMR, Redshift, Kinesis using the search bar
- [ ] Find your IAM user and read your attached policy
- [ ] Search for IAM roles by your prefix
- [ ] Open CloudWatch and find the Log Groups section

---

## Key Concepts Summary

| Service | Category | GCP Equivalent (rough) |
|---|---|---|
| S3 | Object Storage | Cloud Storage |
| Glue | ETL / Data Catalog | Dataflow + Data Catalog |
| Athena | Serverless SQL | BigQuery (ad-hoc) |
| EMR | Managed Spark/Hadoop | Dataproc |
| Redshift | Data Warehouse | BigQuery (warehouse) |
| Kinesis Data Streams | Streaming Ingest | Pub/Sub |
| Amazon Data Firehose | Stream Delivery (S3/Redshift) | Pub/Sub + Dataflow |
| Amazon Managed Service for Apache Flink | Stream Processing | Dataflow |
| DMS | DB Migration | Database Migration Service |
| MWAA | Managed Airflow | Cloud Composer |
| CloudWatch | Monitoring & Logs | Cloud Monitoring + Logging |
| IAM | Identity & Access | IAM |
| VPC | Virtual Networking | VPC |

---

*Next: [Demo — Billing and Cost Explorer](demo-billing-cost-explorer.md)*  
*Tomorrow: [Day 15 — IAM Lab](../day15/lab-iam-explore-and-verify.md)*
