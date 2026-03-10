# Day 15 Demo: IAM Role Architecture and Service Roles

**Type:** Instructor Demo  
**Duration:** ~60 minutes  
**Audience:** Attendees observe while instructor screen-shares  
**Who runs this:** Instructor (requires IAM admin access)

> **Why demo-only?**  
> Attendee IAM policies include an explicit **Deny** on all IAM write actions (`iam:Create*`, `iam:Put*`, `iam:Attach*`, `iam:Detach*`, `iam:Update*`, `iam:Tag*`).  
---

## Instructor Checklist

```bash
export AWS_PROFILE=aws-de-lab
aws sts get-caller-identity
# Must return Account: "<ACCOUNT_ID>"
```

Navigate to the **IAM Console** → confirm you have `AdministratorAccess` or equivalent.

---

## Part 1: IAM Concepts Introduction (15 min)

Walk through these concepts before opening the console.

### IAM Building Blocks

| Concept | Definition | Analogy |
|---|---|---|
| **IAM User** | A person or application with long-term credentials | A person with a badge |
| **IAM Group** | Collection of users sharing a policy | A department with shared access |
| **IAM Role** | An identity assumed by AWS services (not users) | A job title with specific permissions |
| **IAM Policy** | JSON document defining allowed/denied actions | An access control list |
| **Trust Policy** | Defines who/what can assume a role | "Who is allowed to wear this badge" |
| **Inline Policy** | Policy embedded inside a role | Rules printed on the badge |
| **Managed Policy** | Standalone reusable policy | A template badge ruleset |

### Why Roles Instead of Access Keys for Services?

```
BAD:  Hardcode AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY in a Glue job
GOOD: Assign a role → AWS injects temporary credentials automatically
```

A role provides:
- **No long-term secrets** to rotate or leak
- **Short-lived credentials** (STS tokens, 1–12 hrs)
- **Least privilege** — the role only has what that service needs

### Trust Policy vs Permission Policy

```
Trust Policy  → answers: "Who can assume this role?"
                Example: "glue.amazonaws.com may assume this role"

Permission Policy → answers: "What can the role do once assumed?"
                   Example: "Read from s3://trainee01-raw-bucket"
```

---

## Part 2: Roles Created for the Lab — Architecture Overview

Show this diagram on screen (or draw on whiteboard):

```
Glue Service Role
  trust:  glue.amazonaws.com
  perms:  S3 read/write (raw, transformed, scripts, logs)
          Glue catalog full access
          CloudWatch Logs write
          iam:PassRole (to pass itself)

EMR Service Role
  trust:  elasticmapreduce.amazonaws.com
  perms:  AmazonEMRServicePolicy_v2 (managed)
          EC2 Describe*/mutate (supplement)
          iam:PassRole → emr-ec2-role

EMR EC2 Role + Instance Profile
  trust:  ec2.amazonaws.com
  perms:  S3 read/write (raw, transformed, scripts, logs)
          CloudWatch Logs write

Redshift S3 Role
  trust:  redshift.amazonaws.com
  perms:  S3 GetObject on raw/transformed
          Glue GetDatabase/GetTable (for Spectrum)
          iam:PassRole (to pass itself)

Lake Formation Role
  trust:  lakeformation.amazonaws.com
  perms:  S3 read/write (raw, transformed)
          Glue GetDatabase/GetTable/UpdateTable

Amazon Data Firehose Role
  trust:  firehose.amazonaws.com
          (Amazon Data Firehose; formerly Kinesis Data Firehose — trust principal unchanged)
  perms:  S3 PutObject on firehose bucket
          Glue GetTable (for schema conversion)
          Kinesis GetRecords/GetShardIterator
```

### Key design decisions to explain

1. **No `Action: "*"` + `Resource: "*"` together** — every policy is scoped
2. **`iam:PassRole` is always explicit** — AWS API requires it; cannot be inferred
3. **Two separate EMR roles** — service role (cluster management) vs EC2 role (worker node data access). These are never swapped.
4. **S3 bucket ARNs on Day 15** — storage buckets are not yet created. The inline policies today reference placeholder ARNs. After Day 16, the instructor updates the policies with real S3 bucket ARNs.

---

## Part 3: Walk Through Role Creation in the Console (15 min)

Demonstrate creating the Glue service role manually in the IAM console. This walkthrough shows attendees exactly what an IAM role looks like at each step — even though the lab roles were pre-provisioned, every attendee will create roles like this in their future AWS work.

### 3.1 Create a new role — trust relationship

1. **IAM Console** → **Roles** → **Create role**
2. Trusted entity type: **AWS service**
3. Use case: scroll down and select **Glue** → click **Next**

**Explain what just happened:**  
You selected the **trust policy** — the part that answers *"who is allowed to assume this role?"*  
Glue will receive temporary credentials automatically each time it starts a job.

Show the generated trust policy JSON:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "glue.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### 3.2 Attach the base managed policy

4. Search for `AWSGlueServiceRole` → check the box → click **Next**

**Explain:**  
`AWSGlueServiceRole` is an AWS-managed policy that gives Glue baseline access to EC2, CloudWatch Logs, and S3 helper actions. It is maintained by AWS — you do not need to update it when new Glue features ship.

### 3.3 Add an inline policy for S3 access

5. Role name: `trainee01-<BATCH_ID>-glue-service-role` → **Create role**
6. Click into the newly created role → **Add permissions** → **Create inline policy**
7. Switch to **JSON** tab and paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3DataAccess",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::trainee01-<BATCH_ID>-raw",
        "arn:aws:s3:::trainee01-<BATCH_ID>-raw/*",
        "arn:aws:s3:::trainee01-<BATCH_ID>-transformed",
        "arn:aws:s3:::trainee01-<BATCH_ID>-transformed/*"
      ]
    },
    {
      "Sid": "AllowPassRole",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/trainee01-<BATCH_ID>-glue-service-role"
    }
  ]
}
```

**Explain the two statements:**
- `S3DataAccess` — scoped to **specific bucket ARNs**, not `*`. This is least privilege.
- `AllowPassRole` — required any time you tell AWS to *"use this role for a Glue job"*. Scoped to the role's own ARN — never `Resource: "*"`.

### 3.4 EMR — two-role model (explain, do not create)

Explain the two-role EMR pattern (no need to create these — they are already provisioned):

| Role | Trust principal | Purpose |
|---|---|---|
| `traineeNN-<BATCH_ID>-emr-service-role` | `elasticmapreduce.amazonaws.com` | **Cluster controller** — spins up/down EC2 instances, configures networking, manages SGs |
| `traineeNN-<BATCH_ID>-emr-ec2-role` | `ec2.amazonaws.com` | **Worker nodes** — reads/writes S3, pushes logs to CloudWatch |
| `traineeNN-<BATCH_ID>-emr-instance-profile` | wraps `emr-ec2-role` | **Instance profile** — the mechanism that attaches the EC2 role to the actual EC2 instances in the cluster |

**Key point:** The instance profile is a container for one IAM role. EC2 instances cannot directly use an IAM role — they need an instance profile wrapper.

---

## Part 4: Confirm All Attendee Roles Are Provisioned (10 min)

All 22 attendee role sets are pre-provisioned by the instructor before Day 15 starts. Use the CLI to confirm all roles exist:

```bash
export AWS_PROFILE=aws-de-lab

# List all lab roles
aws iam list-roles \
  --query "Roles[?contains(RoleName, '<BATCH_ID>')].[RoleName]" \
  --output table | head -50
```

You should see 6 roles × 22 attendees = **132 role entries** total.

**Per attendee, the following 15 IAM resources exist:**

> Each attendee's resources are isolated by prefix (e.g. `trainee01-<BATCH_ID>-*`). No attendee can see or modify another's roles — their IAM user policy restricts all IAM write actions.

### IAM resources per attendee

| Resource | Type |
|---|---|
| `glue_service_role` | `aws_iam_role` |
| `glue_inline` | `aws_iam_role_policy` |
| `glue_managed` | `aws_iam_role_policy_attachment` |
| `emr_service_role` | `aws_iam_role` |
| `emr_service_managed` | `aws_iam_role_policy_attachment` |
| `emr_service_ec2_supplement` | `aws_iam_role_policy` |
| `emr_ec2_role` | `aws_iam_role` |
| `emr_ec2_inline` | `aws_iam_role_policy` |
| `emr_ec2` | `aws_iam_instance_profile` |
| `redshift_s3_role` | `aws_iam_role` |
| `redshift_inline` | `aws_iam_role_policy` |
| `lakeformation_role` | `aws_iam_role` |
| `lakeformation_inline` | `aws_iam_role_policy` |
| `firehose_role` | `aws_iam_role` |
| `firehose_inline` | `aws_iam_role_policy` |

---

## Part 5: Verify in the Console (5 min)

1. Navigate to **IAM → Roles**
2. Search for `trainee01`
3. Show `trainee01-<BATCH_ID>-glue-service-role`
4. Click the role → **Trust relationships** tab → show `glue.amazonaws.com` trust
5. Click **Permissions** tab → show inline policy + managed policy attachment
6. Click the inline policy → **JSON** → walk through the S3 ARNs
7. Point out: "The S3 ARNs in the Glue and EMR policies currently use placeholder values. After Day 16 (when the S3 buckets are created), the instructor updates these inline policies in the console to reference the real bucket ARNs."

---

## Part 6: IAM Best Practices Summary (5 min)

| Practice | Applied in This Lab |
|---|---|
| **Least Privilege** | Every role scoped to specific actions + ARNs |
| **No Hardcoded Credentials** | Roles replace access keys for services |
| **Separate Roles per Service** | Glue, EMR, Redshift, LakeFormation, Firehose each have distinct roles |
| **Explicit PassRole** | Never rely on implicit pass — always a named statement |
| **Resource-level Scoping** | S3 ARNs specify exact bucket + prefix where possible |
| **Managed Policies for Base Access** | `AWSGlueServiceRole`, `AmazonEMRServicePolicy_v2` for AWS-maintained base permissions |
| **Inline Policies for Custom Permissions** | Service-specific S3/Logs access kept in inline policies |
| **No Action `*` + Resource `*` Together** | Enforced by code review and checkov scan |

---

## Troubleshooting

### Role already exists from a previous batch

If a previous batch's roles were not deleted, remove them manually:

1. **IAM Console** → **Roles** → search for the old batch prefix
2. Select each role → **Delete**
3. Confirm deletion

Alternatively, use the CLI:

```bash
export AWS_PROFILE=aws-de-lab

# List roles from a previous batch ID (e.g. old-batch-id)
aws iam list-roles \
  --query "Roles[?contains(RoleName, 'old-batch-id')].[RoleName]" \
  --output text | while read role; do
    echo "Deleting: $role"
    # Detach managed policies first
    aws iam list-attached-role-policies --role-name "$role" \
      --query 'AttachedPolicies[*].PolicyArn' --output text | \
      tr '\t' '\n' | while read arn; do
        aws iam detach-role-policy --role-name "$role" --policy-arn "$arn"
      done
    # Delete inline policies
    aws iam list-role-policies --role-name "$role" \
      --query 'PolicyNames[*]' --output text | \
      tr '\t' '\n' | while read p; do
        aws iam delete-role-policy --role-name "$role" --policy-name "$p"
      done
    aws iam delete-role --role-name "$role"
  done
```

### Placeholder ARNs after Day 16

After the Day 16 S3 buckets are provisioned, update the inline policies for each attendee's Glue and EMR EC2 roles:

1. **IAM Console** → **Roles** → open `traineeNN-<BATCH_ID>-glue-service-role`
2. Click the inline policy → **Edit**
3. Replace placeholder ARNs with real bucket ARNs (format: `arn:aws:s3:::traineeNN-<BATCH_ID>-raw`)
4. Click **Save changes**
5. Repeat for each attendee and each affected role

---

*Attendees: continue to [Lab — Explore and Verify IAM Roles](lab-iam-explore-and-verify.md)*  
*Next demo: [Billing and Budget Alerts](demo-billing-budget-alerts.md)*
