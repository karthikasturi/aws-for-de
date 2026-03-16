# Day 20 Demo: AWS Batch — Managed Batch Computing

**Type:** Instructor Demo  
**Duration:** ~30 minutes  
**Day:** 20 — Real-Time Streaming and Batch Processing  
**Why demo-only:** AWS Batch is not in the lab Terraform infrastructure. Creating compute environments requires `batch:CreateComputeEnvironment` + `ec2:*` + IAM role creation — outside attendee scope.

---

## Instructor Checklist

**Pre-demo environment is pre-provisioned.** The following resources already exist:

| Resource | Name |
|---|---|
| Compute Environment | `aws-de-lab-batch-demo-ce` (MANAGED / FARGATE) |
| Job Queue | `aws-de-lab-batch-demo-queue` (priority 100) |
| Job Definition | `taxi-parquet-converter:3` (python:3.11-slim, 0.25 vCPU / 512 MiB) |
| Task Execution Role | `aws-de-lab-batch-task-execution-role` |
| Job Role | `aws-de-lab-batch-job-role` (S3 read on raw buckets) |

```bash
export AWS_PROFILE=aws-de-lab
aws sts get-caller-identity   # Must return Account: "985539779502"

# Confirm demo environment is VALID before class starts
aws batch describe-compute-environments \
  --compute-environments aws-de-lab-batch-demo-ce \
  --region ap-south-1 \
  --query 'computeEnvironments[0].{State:state,Status:status}'
# Expected: State=ENABLED, Status=VALID

aws batch describe-job-queues \
  --job-queues aws-de-lab-batch-demo-queue \
  --region ap-south-1 \
  --query 'jobQueues[0].{State:state,Status:status}'
# Expected: State=ENABLED, Status=VALID
```

Console: **AWS Batch** (ap-south-1)

---

## Part 1: What Is AWS Batch? (10 min)

### The problem AWS Batch solves

```
Without Batch — managing compute yourself:
  1. Provision EC2 instances (or ECS cluster)
  2. Write job scheduler (which node runs which job?)
  3. Handle node failures and job retries
  4. Scale down when queue is empty
  5. Monitor job state

With AWS Batch:
  1. Define what to run (Docker container + command)
  2. Define your compute budget (instance types, min/max vCPUs)
  3. Submit job
  4. AWS Batch handles: scheduling, scaling, retries, monitoring
```

### Batch vs Lambda vs EMR vs Glue

| | AWS Batch | Lambda | EMR | Glue |
|---|---|---|---|---|
| **Max runtime** | No limit | 15 minutes | No limit (cluster lifetime) | 30 min (lab), 2880 min default |
| **Container** | Any Docker image | No (managed runtime) | No (YARN/Spark) | Managed Spark |
| **Compute** | EC2 or Fargate | Serverless | EC2 (managed) | Serverless Spark |
| **State** | Job queue + job definitions | Stateless | Cluster steps | Job runs |
| **Best for** | Long-running batch jobs, HPC | Event-driven under 15 min | Large-scale Spark/Hadoop | Spark ETL without cluster management |
| **Pricing** | Pay for EC2/Fargate usage | Per-invocation | Per instance-hour | Per DPU-hour |

---

## Part 2: AWS Batch Architecture (5 min)

```
                    ┌─────────────────────────────────────────────────────┐
                    │              AWS Batch                               │
                    │                                                      │
  Job             ┌─┴──────────────┐         ┌────────────────────────┐   │
  Submission ────►│  Job Queue     │────────►│  Compute Environment   │   │
                  │                │         │                         │   │
                  │  Priority 1:   │         │  Managed: Batch creates │   │
                  │  high-priority │         │  EC2 (On-Demand/Spot)   │   │
                  │                │         │  or Fargate tasks       │   │
                  │  Priority 2:   │         │                         │   │
                  │  standard      │         │  minvCPUs: 0            │   │
                  │                │         │  maxvCPUs: 256          │   │
                  └─┬──────────────┘         │  (scales to 0 at idle)  │   │
                    │                        └────────────────────────┘   │
                    │            ┌──────────────────────────────────────┐  │
                    │            │  Job Definition                      │  │
                    │            │  - Docker image: my-etl:latest       │  │
                    │            │  - Command: python process.py        │  │
                    │            │  - vCPUs: 2, Memory: 4 GiB           │  │
                    │            │  - Retry attempts: 3                 │  │
                    │            │  - Environment vars: S3_BUCKET=…     │  │
                    │            └──────────────────────────────────────┘  │
                    └─────────────────────────────────────────────────────┘
```

---

## Part 3: Compute Environment Types (5 min)

### Managed Compute Environments

AWS Batch provisions and terminates EC2 instances automatically.

**EC2 (On-Demand):**
- Predictable cost
- Any instance type (c5, m5, r5, p3 for GPU)
- Good for SLA-critical jobs that cannot tolerate interruption

**Spot:**
- Up to 90% cost reduction
- Instances can be interrupted (2-minute warning)
- Batch automatically checkpoints and retries interrupted jobs
- Best for fault-tolerant batch workloads

**Fargate:**
- Serverless container execution — no EC2 instance management
- Cold start slower (~1 minute) vs EC2 (already warm)
- Good for occasional jobs under 4 vCPU / 30 GiB

### Demo: Show compute environment in console

1. Console → **AWS Batch** → **Compute environments**
2. Click `aws-de-lab-batch-demo-ce`:
   - **Type:** Managed / Fargate
   - **State:** ENABLED, **Status:** VALID
   - **Max vCPUs:** 16 (scales to 0 — no cost when idle)
   - **Subnets:** 3 default-VPC subnets across 3 AZs
3. Note: no EC2 instances listed — Fargate is serverless, Batch launches tasks only when jobs are submitted

---

## Part 4: Demo — Create a Job Definition (10 min)

### Demo: Walk through Job Definition creation

1. Console → **AWS Batch** → **Job definitions**
2. Click `taxi-parquet-converter` revision 3 — walk through the fields:
   - **Platform:** Fargate
   - **Container image:** `public.ecr.aws/docker/library/python:3.11-slim`
   - **Command:** `sh -c 'pip install boto3 -q && python3 -c "import boto3, os; print(...)"'`
   - **vCPUs:** 0.25, **Memory:** 512 MiB
   - **Job role:** `aws-de-lab-batch-job-role` (S3 read on raw bucket)
   - **Execution role:** `aws-de-lab-batch-task-execution-role` (pulls image, writes logs)
3. Show **Environment variables** section:
   - `SOURCE_BUCKET=trainee01-2026-03-raw` — job discovers which bucket to read
   - `TARGET_BUCKET=trainee01-2026-03-transformed`
4. Show **Retry strategy** — 3 attempts
5. Show **Timeout** — 3600 seconds (job killed if it exceeds this)

### A real ETL job definition in code

```json
{
  "jobDefinitionName": "taxi-parquet-converter",
  "type": "container",
  "platformCapabilities": ["FARGATE"],
  "containerProperties": {
    "image": "public.ecr.aws/docker/library/python:3.11-slim",
    "command": ["sh", "-c",
      "pip install boto3 -q && python3 convert.py"
    ],
    "resourceRequirements": [
      {"type": "VCPU",   "value": "2"},
      {"type": "MEMORY", "value": "4096"}
    ],
    "jobRoleArn":       "arn:aws:iam::985539779502:role/aws-de-lab-batch-job-role",
    "executionRoleArn": "arn:aws:iam::985539779502:role/aws-de-lab-batch-task-execution-role",
    "environment": [
      {"name": "SOURCE_BUCKET", "value": "trainee01-2026-03-raw"},
      {"name": "TARGET_BUCKET", "value": "trainee01-2026-03-transformed"}
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group":  "/aws/batch/job",
        "awslogs-region": "ap-south-1"
      }
    },
    "networkConfiguration": {"assignPublicIp": "ENABLED"}
  },
  "retryStrategy": {"attempts": 3},
  "timeout": {"attemptDurationSeconds": 3600}
}
```

> **Note:** Use `public.ecr.aws/docker/library/python:3.11-slim`, **not** `amazonlinux:2023` — Amazon Linux doesn't include Python or boto3 by default.

---

## Part 5: Job Submission and Monitoring (5 min)

### Submitting a job via CLI

```bash
# Submit a live demo job against the pre-provisioned queue + definition
aws batch submit-job \
  --job-name taxi-etl-$(date +%Y%m%d-%H%M%S) \
  --job-queue aws-de-lab-batch-demo-queue \
  --job-definition taxi-parquet-converter \
  --region ap-south-1 \
  --query '{JobId:jobId,JobName:jobName}'

# Override env vars for a specific source prefix
aws batch submit-job \
  --job-name taxi-etl-override \
  --job-queue aws-de-lab-batch-demo-queue \
  --job-definition taxi-parquet-converter \
  --container-overrides '{
    "environment": [
      {"name": "SOURCE_PREFIX", "value": "taxi/2024/01/15/"}
    ]
  }' \
  --region ap-south-1
```

### Check job status and logs

```bash
# Describe a submitted job (replace JOB_ID)
aws batch describe-jobs \
  --jobs JOB_ID \
  --region ap-south-1 \
  --query 'jobs[0].{Status:status,Reason:statusReason,Log:container.logStreamName,ExitCode:container.exitCode}'

# Read CloudWatch logs once job has a logStreamName
aws logs get-log-events \
  --log-group-name /aws/batch/job \
  --log-stream-name "LOG_STREAM_NAME" \
  --region ap-south-1 \
  --query 'events[*].message' --output text
```

> **Live demo output (verified 2026-03-16):**
> ```
> Job running OK. boto3=1.42.68 SOURCE=trainee01-2026-03-raw
> ```

### Job lifecycle states

```
SUBMITTED → PENDING → RUNNABLE → STARTING → RUNNING → SUCCEEDED
                                                      ↘ FAILED
```

- **SUBMITTED:** Job received, not yet assigned to queue
- **PENDING:** Waiting for a priority slot in the queue
- **RUNNABLE:** Queue accepted the job, waiting for compute capacity
- **STARTING:** Instance provisioning / container pulling
- **RUNNING:** Container executing

```bash
# Describe a job (replace with your actual JOB_ID from submit-job output)
aws batch describe-jobs \
  --jobs JOB_ID \
  --region ap-south-1 \
  --query 'jobs[0].{Status:status,StartedAt:startedAt,StoppedAt:stoppedAt,LogStream:container.logStreamName}'
```

---

## Part 6: Batch for Data Engineering Use Cases (5 min)

### Common patterns

**Pattern 1: Daily ETL trigger**
```
EventBridge Scheduler (daily at 02:00)
  → Lambda (submit Batch job)
  → Batch Job (read yesterday's S3 data, transform, write Parquet)
  → SNS notification on success/failure
```

**Pattern 2: File-triggered processing**
```
S3 upload event
  → EventBridge rule (matches s3:ObjectCreated on raw bucket)
  → Batch job (process that specific file, write to transformed)
```

**Pattern 3: Array jobs (parallel processing)**
```
Single Batch job with arrayProperties: {size: 100}
  → Creates 100 child jobs, each gets AWS_BATCH_JOB_ARRAY_INDEX env var (0-99)
  → Each child processes one date partition
  → Reduces 100-hour sequential job to ~1.5-hour parallel job
```

### Batch vs Glue for your data pipeline

| Scenario | Better choice |
|---|---|
| Transform S3 CSV → Parquet (standard PySpark ETL) | AWS Glue |
| Long-running ML training job (> 30 min) | AWS Batch |
| Custom Docker container with proprietary library | AWS Batch |
| GUI-driven visual ETL design | AWS Glue Studio |
| HPC job array (100+ parallel tasks) | AWS Batch |
| Serverless, no cluster management, standard Spark | AWS Glue |

---

## Key Takeaways

| Concept | Rule of thumb |
|---|---|
| **Scale to zero** | Set `minvCPUs = 0` — you pay nothing when queues are empty |
| **Spot for savings** | Use Spot for fault-tolerant batch jobs — 70–90% cheaper |
| **Array jobs** | Use for massively parallel independent tasks (date partitions, file processing) |
| **Container-first** | Any language, any library — if it runs in Docker, it runs on Batch |
| **vs Glue** | Glue is Batch-like for Spark ETL specifically; Batch is general-purpose |

---

*Attendees: return to [Lab — Kinesis Streaming](lab-kinesis.md)*  
*Tomorrow: [Day 21 — EMR PySpark Lab](../day21/lab-emr-pyspark.md)*
