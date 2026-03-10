# Day 15 Demo: Billing Alerts and Budget Management

**Type:** Instructor Demo  
**Duration:** ~20 minutes  
**Audience:** Attendees observe while instructor screen-shares  
**Who runs this:** Instructor (requires `budgets:*` and `billing:*` account-level access)

> **Why demo-only?**  
> Creating and managing billing alerts and budgets requires account-level billing access (`budgets:ModifyBudget`, `ce:CreateCostCategoryDefinition`, `aws-portal:ModifyBilling`).  
> These permissions are not granted to attendee IAM users — they are account-wide and cannot be safely scoped by prefix.  
> The instructor creates a **shared lab budget** that monitors total training spend.

---

## Instructor Checklist

```bash
export AWS_PROFILE=aws-de-lab
aws sts get-caller-identity
# Must return Account: "<ACCOUNT_ID>"
```

Confirm billing console access: navigate to the console → search `Billing` → Billing and Cost Management. If you see a "No access" error, switch to your admin-level profile before proceeding.

---

## Part 1: Why Billing Alerts Matter in a Training Lab (3 min)

Before opening the console, explain the cost risks specific to this lab:

| Risk | Worst-case cost | Guard in place |
|---|---|---|
| Glue job with default timeout | 2880 min × Glue DPU rate = potentially $100s (check [ap-south-1 Glue pricing](https://aws.amazon.com/glue/pricing/)) | A 30-minute timeout is set on every Glue job |
| EMR cluster left running overnight | m5.xlarge × 2 nodes × hourly rate × 8 hr (check [EC2 ap-south-1 pricing](https://aws.amazon.com/ec2/pricing/on-demand/)) | Auto-termination configured: clusters shut down after 2 hours idle |
| Redshift cluster left running | ra3.large × 24 hr (check [ap-south-1 Redshift pricing](https://aws.amazon.com/redshift/pricing/)) | Instructor tears down day-specific resources each evening |
| RDS instance left running | db.t3.micro × 24 hr (check [ap-south-1 RDS pricing](https://aws.amazon.com/rds/mysql/pricing/)) | Instructor tears down RDS on Day 17 evening |

> "Budgets are the safety net for the rare case where a resource escapes the nightly teardown, or a new attendee accidentally creates an unscoped resource."

---

## Part 2: Open the Billing Dashboard (3 min)

**Navigate:** Console search → `Billing` → Billing and Cost Management

### What to show

1. **Billing overview** page — current month estimated charges
2. Highlight **Service charges** table — sorted by cost
   - For a day-1 lab: should be near $0
   - For an in-progress lab: show EC2 (workstations), any running Redshift/RDS
3. Point out the **Cost by tag** section — scroll to find `Batch: <BATCH_ID>` and `Environment: training`

**Talking point:**
> "Notice how the tag filter immediately isolates our training lab spend from everything else in the account. This is exactly why tagging is a first-class concern, not an afterthought."

---

## Part 3: Create a Lab Budget (8 min)

**Navigate:** Billing → Budgets → Create budget

### Step 1: Budget setup method

Select: **Use a template (simplified)**

Choose template: **Monthly cost budget**

### Step 2: Budget details

| Field | Value | Explanation |
|---|---|---|
| **Budget name** | `aws-de-lab-<BATCH_ID>-monthly` | Descriptive, includes batch ID |
| **Budgeted amount** | `$150` | Estimated maximum for 7-day lab with 22 attendees |
| **Email recipients** | Instructor email(s) | Comma-separated for multiple |

### Step 3: Alert thresholds

The template includes a default 85% threshold. Show attendees how to add more:

Click **Add alert threshold** to add:

| Alert | Threshold | Type | Meaning |
|---|---|---|---|
| Alert 1 | 60% | Actual | $90 — heads-up, lab on track |
| Alert 2 | 85% | Actual | $127.50 — warning, check for runaway resources |
| Alert 3 | 100% | Actual | $150 — budget hit, investigate immediately |
| Alert 4 | 115% | Forecasted | Projected to exceed $172.50 — consider early teardown |

### Step 4: Create the budget

Click **Create budget**.

After creation, show the **Budgets dashboard**:
- Forecasted spend bar
- Actual vs budgeted comparison
- Alert status indicators (green until threshold hit)

---

## Part 4: Set Up a Cost Anomaly Detection Alert (4 min)

**Navigate:** Billing → Cost Anomaly Detection → Create monitor

> Cost Anomaly Detection uses ML to flag unusual spending patterns, even within budget. It's a complement to fixed-amount budgets.

### Step-by-step

**Step 1:** Click **Create monitor**

**Step 2:** Monitor type: **AWS services**

**Step 3:** Monitor name: `aws-de-lab-anomaly-monitor`

**Step 4:** Alert subscription:
- Alert type: **Individual alerts**
- Threshold: `$10` (absolute)
- Email: instructor email

**Step 5:** Click **Create monitor**

**Talking point:**
> "A budget tells you when you've hit a fixed amount. Anomaly detection tells you when spending suddenly spikes relative to your normal pattern — e.g., if someone accidentally runs 20 EMR clusters instead of 1. Both are needed."

---

## Part 5: Activate Cost Allocation Tags (2 min)

**Navigate:** Billing → Cost allocation tags → User-defined tags

### What to show

Every resource in the lab is tagged with three standard tags at creation time:

| Tag key | Value |
|---|---|
| `Environment` | `training` |
| `Batch` | `<BATCH_ID>` |
| `ManagedBy` | `lab` |

For these tags to appear as filterable dimensions in Cost Explorer and Budgets, they must be **activated** here in the Billing console.

1. Check that `Environment`, `Batch`, and `ManagedBy` show as **Active**
2. If any show as **Inactive**: click the checkbox → **Activate**

> Note: After activation it can take up to 24 hours for historical data to become filterable. Going forward, all new cost data will be tagged immediately.

---

## Part 6: Budget Alert Workflow — What Happens When an Alert Fires

Walk attendees through the response process:

```
1. Budget threshold hit
       ↓
2. AWS sends email to instructor
       ↓
3. Instructor checks Cost Explorer:
   - Filter by tag: Environment = training
   - Group by: Service
   - Find the unexpected spike
       ↓
4. Find stray resources via the console:
   - Tag Editor (Resource Groups) → search tag Environment=training
   - Compare against expected resources for the current lab day
       ↓
5. Stop or delete the offending resource:
   - For EC2/EMR: terminate from EC2 or EMR console
   - For Redshift/RDS: delete cluster/instance from console
   - For S3 objects causing costs: empty the bucket via S3 console
```

---

## Part 7: Key Billing Concepts Summary

| Concept | Description |
|---|---|
| **AWS Budgets** | Set monthly/quarterly spend limits with email/SNS/Lambda actions |
| **Cost Explorer** | Interactive cost analysis by service, tag, account, region, time |
| **Cost Anomaly Detection** | ML-based detection of spending spikes outside normal patterns |
| **Cost Allocation Tags** | Dimensions for cost attribution — must be activated in Billing console |
| **Savings Plans** | Committed usage discounts (compute, EC2, SageMaker) |
| **Reserved Instances** | Upfront pricing for steady-state EC2/RDS/Redshift workloads |
| **rightsizing** | Recommendations to downsize over-provisioned instances |

---

## What Attendees Can Do (Read-Only Access)

Even without billing write access, attendees can do the following from the CLI:

```bash
# View existing budgets (read-only)
aws budgets describe-budgets --account-id <ACCOUNT_ID>

# View current month cost by service (requires ce:GetCostAndUsage)
# Note: ce:* is not in attendee policy — this will fail as expected
aws ce get-cost-and-usage \
  --time-period Start=<BATCH_ID>-01,End=<BATCH_ID>-31 \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE
```

> The `AccessDenied` on the Cost Explorer command is expected — it reinforces the principle that billing data is account-wide sensitive information, not per-resource scoped.

---

## Nightly Teardown Reminder

Before ending each lab day, the instructor manually deletes day-specific resources from the AWS console or CLI:

```bash
export AWS_PROFILE=aws-de-lab

# Day 15 — IAM roles stay (needed for all subsequent days)
# Day 16 — S3 buckets stay (needed for all subsequent days)
# Day 17 evening — delete each attendee's RDS instance:
aws rds delete-db-instance \
  --db-instance-identifier traineeNN-<BATCH_ID>-rds \
  --skip-final-snapshot

# Day 18 evening — delete each attendee's Redshift cluster:
aws redshift delete-cluster \
  --cluster-identifier traineeNN-<BATCH_ID>-redshift \
  --skip-final-cluster-snapshot

# Day 20 evening — delete Kinesis streams and Firehose:
aws kinesis delete-stream --stream-name traineeNN-<BATCH_ID>-stream
aws firehose delete-delivery-stream --delivery-stream-name traineeNN-<BATCH_ID>-firehose
```

> Repeat each CLI command for all 22 attendees, or run a loop: `for n in $(seq -w 1 22); do aws ... trainee${n}-...; done`

---

*Back to: [Lab — Explore and Verify IAM Roles](lab-iam-explore-and-verify.md)*  
*Tomorrow: [Day 16 — S3 and KMS Storage](../day16/)*
