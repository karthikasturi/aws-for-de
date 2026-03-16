# Day 21 Demo: EMR Advanced — Cluster Creation, EMR Studio, and Auto-Termination

**Type:** Instructor Demo  
**Duration:** ~45 minutes  
**Day:** 21 — Big Data Processing with EMR  
**Why demo-only:** Creating EMR clusters requires `ec2:RunInstances`, `iam:PassRole`, and broad EC2 permissions that are outside attendee prefix-scoped policy. EMR Studio creation requires additional IAM and VPC setup.

---

## Instructor Checklist

```bash
export AWS_PROFILE=aws-de-lab
aws sts get-caller-identity   # Must return Account: "<ACCOUNT_ID>"
```

Console: **Amazon EMR**

---

## Part 1: EMR Cluster Architecture Recap (5 min)

### Node roles

```
                    ┌─────────────────────────────────────────────────────┐
                    │              EMR Cluster (YARN + Spark)              │
                    │                                                      │
                    │  Master Node (1)                                    │
                    │   - YARN ResourceManager                            │
                    │   - Spark Driver (cluster mode)                     │
                    │   - Hive Metastore / Glue catalog client            │
                    │   - EMR Step coordinator                            │
                    │                                                      │
                    │  Core Nodes (1+)                                    │
                    │   - YARN NodeManager                                │
                    │   - HDFS DataNode (ephemeral — lost on termination) │
                    │   - Spark Executors                                 │
                    │                                                      │
                    │  Task Nodes (0+, optional)                         │
                    │   - YARN NodeManager only (no HDFS)                │
                    │   - Ideal for Spot — can be removed anytime        │
                    └─────────────────────────────────────────────────────┘
```

> **Key insight for data engineering on AWS:** Never rely on HDFS for persistent storage. Always read from and write back to S3. Use HDFS only for intermediate shuffle data within a job. This way, when the cluster terminates, you lose nothing.

---

## Part 2: Demo — Create an EMR Cluster (15 min)

### Walk through console cluster creation

1. Console → **Amazon EMR** → **Create cluster**
2. Walk through each section:

**Application bundle:**
```
Spark (Spark 3.4, includes Hadoop, Hive, YARN)
  — Good for: pure Spark ETL/analytics
  
Spark + HBase
  — Good for: real-time random-access key-value + batch

Spark + Presto
  — Good for: SQL analytics in addition to batch processing
```

**Cluster configuration:**
```
Release:   emr-6.15.0
Node types:
  Master:  m5.xlarge  (1)
  Core:    m5.xlarge  (1)
  
(For larger jobs, demonstrate adding a Task Instance Group with Spot)
```

**Networking:**
```
VPC:       [lab VPC]
Subnet:    private subnet in ap-south-1a
Security:  traineeNN-2026-03-emr-master (master)
           traineeNN-2026-03-emr-slave  (core + task)
```

**Security:**
```
EC2 key pair:        (lab key or none for step-only usage)
IAM service role:    traineeNN-2026-03-emr-service-role
IAM instance profile: traineeNN-2026-03-emr-instance-profile
```

**Logging:**
```
S3 log URI:  s3://traineeNN-2026-03-logs/emr/
```

**Auto-termination:**
```
Idle timeout: 7200 seconds (2 hours)
— Cluster terminates automatically after 2 hours of no RUNNING steps
— CRITICAL for cost control — an idle m5.xlarge costs ~$0.23/hr, 24/7 ≈ $168/month
```

**Bootstrap actions:**
```
s3://traineeNN-2026-03-scripts/emr/bootstrap.sh
— Runs on every node before Spark starts
— Used to: install Python packages, configure OS settings, pre-download data
```

3. Show the cost estimate at the bottom
4. **Cancel** — the cluster for the lab was already created earlier

### Create via CLI (show, don't run)

```bash
# This is what Terraform generated for the lab cluster (simplified)
aws emr create-cluster \
  --name "traineeNN-2026-03-cluster" \
  --release-label emr-6.15.0 \
  --applications Name=Spark Name=Hadoop \
  --ec2-attributes '{
    "SubnetId": "subnet-xxxxxxxxxxxxxxxxx",
    "EmrManagedMasterSecurityGroup": "sg-master-id",
    "EmrManagedSlaveSecurityGroup":  "sg-slave-id",
    "InstanceProfile": "traineeNN-2026-03-emr-instance-profile"
  }' \
  --instance-groups '[
    {
      "Name": "Master",
      "InstanceGroupType": "MASTER",
      "InstanceType": "m5.xlarge",
      "InstanceCount": 1,
      "Market": "ON_DEMAND"
    },
    {
      "Name": "Core",
      "InstanceGroupType": "CORE",
      "InstanceType": "m5.xlarge",
      "InstanceCount": 1,
      "Market": "ON_DEMAND"
    }
  ]' \
  --bootstrap-actions '[{
    "Name": "install-packages",
    "Path": "s3://traineeNN-2026-03-scripts/emr/bootstrap.sh"
  }]' \
  --log-uri "s3://traineeNN-2026-03-logs/emr/" \
  --service-role traineeNN-2026-03-emr-service-role \
  --auto-termination-policy '{"IdleTimeout": 7200}' \
  --region ap-south-1
```

---

## Part 3: Spot Instances with EMR (5 min)

### Instance Groups vs Instance Fleets

**Instance Groups** (used in this lab):
- One node type per group
- All instances in a group are the same instance type
- Spot bid = on-demand price (AWS determines actual spot price)
- Use for predictable workloads

**Instance Fleets** (more flexible):
- Specify a list of acceptable instance types per fleet
- Batch chooses whichever is cheapest and most available
- Better Spot availability — if `m5.xlarge` is scarce, EMR tries `m5.2xlarge` (fewer of them), or `m4.xlarge`

```json
// Instance Fleet for Core nodes (more Spot-resilient)
{
  "Name": "Core Fleet",
  "InstanceFleetType": "CORE",
  "TargetSpotCapacity": 4,
  "InstanceTypeConfigs": [
    {"InstanceType": "m5.xlarge",  "WeightedCapacity": 1},
    {"InstanceType": "m5.2xlarge", "WeightedCapacity": 2},
    {"InstanceType": "m4.xlarge",  "WeightedCapacity": 1}
  ],
  "LaunchSpecifications": {
    "SpotSpecification": {
      "TimeoutDurationMinutes": 20,
      "TimeoutAction": "SWITCH_TO_ON_DEMAND"
    }
  }
}
```

`SWITCH_TO_ON_DEMAND` → if no Spot is available after 20 minutes, automatically use On-Demand instead of failing.

---

## Part 4: EMR Studio (10 min)

### What EMR Studio is

EMR Studio is a managed Jupyter/JupyterLab environment that connects directly to EMR clusters.

```
Data Engineer / Analyst
  │
  │  Browser (HTTPS)
  ↓
EMR Studio (AWS-managed JupyterHub)
  │
  │  Spark kernel
  ↓
EMR Cluster (persistent or serverless)
  │
  ↓
S3 / Glue Catalog
```

**Advantages over self-hosted Jupyter:**
- No port tunneling to cluster master node
- Notebooks stored in S3 (version controlled)
- Multiple kernels: Python 3, PySpark, SparkR
- Git repository integration
- Collaboration: multiple users on same workspace

### Demo: EMR Studio walkthrough

1. Console → **Amazon EMR** → **EMR Studio** → **Create Studio** (show the form, don't create)
2. Key settings:
   - **Authentication:** AWS SSO (recommended) or IAM
   - **VPC and subnets:** must be in same VPC as cluster
   - **S3 location for notebooks:** `s3://traineeNN-2026-03-logs/notebooks/`
   - **Service role:** needs S3 read/write + Glue catalog access

### Demo: Workspace inside Studio (if a studio is pre-created)

1. Open the studio URL → **Create workspace**
2. Attach to the lab cluster `traineeNN-2026-03-cluster`
3. Create a new notebook → select **PySpark** kernel
4. Type and run a cell:

```python
# This runs on the EMR cluster, not locally
from pyspark.sql import functions as F

df = spark.read.parquet("s3://trainee01-2026-03-transformed/taxi_parquet/")
df.groupBy("pickup_year", "pickup_month").count().show()
```

5. Show: output appears in the browser; Spark is running on the cluster

---

## Part 5: EMR Serverless (5 min)

Like Redshift Serverless, EMR Serverless removes cluster management:

| | EMR on EC2 | EMR Serverless |
|---|---|---|
| **Cluster management** | You create/terminate | AWS manages, auto-scales |
| **Idle cost** | Pay for cluster runtime | Scale to zero |
| **Cold start** | 5–10 min (new cluster) | ~15 seconds (pre-initialized workers) |
| **Spot instances** | Yes, configurable | No (AWS manages the fleet) |
| **Custom bootstrap** | Yes | No |
| **Control** | Full (Spark configs, instance types) | Limited |

### Demo: EMR Serverless in console

1. Console → **Amazon EMR** → **EMR Serverless**
2. Click **Create application**:
   - **Type:** Spark
   - **Release:** emr-6.15.0 (same as provisioned)
   - **Initial capacity:** 0 workers (scale to zero)
   - **Max capacity:** 200 vCPU, 1024 GB RAM
3. Show the **Submit job** form:
   - Entry point: the same S3 script path
   - Same `--source-bucket` arguments
4. Cancel — explain: job submission API is nearly identical to EMR on EC2

---

## Part 6: Cost Architecture for EMR Workloads (5 min)

### Cost model

| Component | Billing |
|---|---|
| EC2 instances (master + core) | Per second when running |
| EBS volumes (attached to nodes) | Per GB-month |
| EMR premium (service charge) | ~25% on top of EC2 cost |
| S3 reads/writes | Per request + per GB transferred (typically small) |
| Glue catalog API calls | Per month (first million free) |

### Cost optimization strategies

**1. Auto-termination (lab default: 7200 seconds)**
```
Never leave a cluster WAITING with no steps.
An idle m5.xlarge master + m5.xlarge core = ~$0.45/hr
7200 sec idle timeout prevents overnight charges if someone forgets to terminate.
```

**2. Spot for Core nodes (70–90% savings)**
```
Use Spot for core + task nodes; On-Demand for master only
Set SWITCH_TO_ON_DEMAND as fallback so jobs don't fail on Spot interruption
```

**3. EMR Serverless for infrequent jobs**
```
If your ETL runs once per day for 10 minutes:
  Provisioned cluster: $0.45/hr × 24hr = $10.80/day
  EMR Serverless:      $0.052/vCPU-hr × 4 vCPU × 0.17hr = $0.035/day
  Savings: 99.7%
```

**4. S3 as primary storage (not HDFS)**
```
Terminate cluster after every job run.
Next run: spin up a fresh cluster, read from S3, write to S3, terminate.
No state in the cluster = no "zombie cluster" cost.
```

---

## Key Takeaways

| Concept | Decision point |
|---|---|
| **Master node** | Always On-Demand (it's the cluster coordinator — cannot be interrupted) |
| **Core nodes** | Spot if job can tolerate potential retries; On-Demand for strict SLA |
| **Task nodes** | Always Spot (no HDFS = safe to add/remove at any time) |
| **HDFS** | Never use for persistent data — S3 only |
| **Auto-termination** | Always set `IdleTimeout`; lab requirement is 7200 seconds |
| **EMR Studio** | Use for interactive exploration/dev; switch to steps for production |
| **EMR Serverless** | Use when workloads are infrequent or unpredictable |
| **Glue integration** | `enableHiveSupport()` + `AWSGlueDataCatalogHiveClientFactory` — use the Glue catalog as metastore |

---

*Attendees: return to [Lab — EMR PySpark](lab-emr-pyspark.md)*  
*This completes the lab series (Days 15–21). See [README](../../README.md) for the full guide index.*
