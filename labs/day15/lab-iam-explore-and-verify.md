# Day 15 Lab: Explore and Verify IAM Roles

**Type:** Attendee Self-Service Lab  
**Duration:** ~45 minutes  
**Prerequisites:** Instructor has pre-provisioned all IAM roles for your attendee prefix  
**Skills practiced:** Console navigation, policy reading, CLI credential verification, least-privilege reasoning

> **Note on permissions:** Your attendee policy intentionally **prevents you from creating or modifying IAM resources**.  
> This is a shared account — IAM write access for 25 attendees would be a security incident waiting to happen.  
> In this lab you will explore, inspect, and verify the roles that have been set up for you.

---

## Lab Setup

### Configure your AWS CLI

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
    "Arn": "arn:aws:iam::<ACCOUNT_ID>:user/trainee01"
}
```

> Replace `trainee01` with your actual attendee ID throughout this lab.

---

## Part 1: Inspect Your IAM Roles in the Console

### 1.1 Navigate to IAM Roles

1. Open the AWS Console → search bar → `IAM` → **Roles**
2. In the search box, type your attendee prefix (e.g. `trainee01`)

You should see **6 roles**:

| Role Name | Purpose |
|---|---|
| `trainee01-<BATCH_ID>-glue-service-role` | Glue crawlers and ETL jobs |
| `trainee01-<BATCH_ID>-emr-service-role` | EMR cluster management |
| `trainee01-<BATCH_ID>-emr-ec2-role` | EMR EC2 worker nodes |
| `trainee01-<BATCH_ID>-redshift-s3-role` | Redshift COPY and Spectrum |
| `trainee01-<BATCH_ID>-lakeformation-role` | Lake Formation service |
| `trainee01-<BATCH_ID>-firehose-role` | Amazon Data Firehose delivery |

> If you see **0 roles**, let the instructor know — the role provisioning step has not been completed yet.

### 1.2 Deep-dive: Glue Service Role

1. Click `trainee01-<BATCH_ID>-glue-service-role`
2. Go to the **Trust relationships** tab

**Question 1:** What principal is listed as trusted? What does this mean?

<details>
<summary>Answer</summary>

`glue.amazonaws.com` — only the AWS Glue service can assume this role. No IAM user, no EC2 instance — only Glue. This is the trust policy enforced by STS.

</details>

3. Go to the **Permissions** tab
4. You will see two policies:
   - `AWSGlueServiceRole` — AWS managed policy (click to view in a new tab)
   - `glue-inline-policy` — custom inline policy (click the `{}` JSON icon)

5. In the inline policy JSON, look for:

```json
{
  "Sid": "GlueS3ObjectAccess",
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
  "Resource": [...]
}
```

**Question 2:** What S3 ARNs are listed in the Resource field right now? Why do they look unusual?

<details>
<summary>Answer</summary>

They contain `placeholder-bucket-not-yet-created`. This is a safe fallback value used when the S3 buckets don't exist yet — the roles are provisioned on Day 15, but the S3 buckets are only created on Day 16. After Day 16, the instructor updates the inline policies with the real bucket ARNs.

</details>

**Question 3:** Find the `AllowPassRole` statement. What resource ARN does it point to?

<details>
<summary>Answer</summary>

`arn:aws:iam::<ACCOUNT_ID>:role/trainee01-<BATCH_ID>-glue-service-role` — the role's own ARN. This allows the role to "pass itself" to Glue jobs. Without this, even a valid AWS principal with IAM permissions would get `AccessDenied` when trying to create a Glue job referencing this role.

</details>

---

### 1.3 Deep-dive: EMR Roles

1. Click `trainee01-<BATCH_ID>-emr-service-role`
2. Note the trust: `elasticmapreduce.amazonaws.com`
3. Navigate to **Instance profiles** (left sidebar in IAM)
4. Search for `trainee01` — find `trainee01-<BATCH_ID>-emr-instance-profile`
5. Click it — note it wraps `trainee01-<BATCH_ID>-emr-ec2-role`

**Question 4:** Why does EMR need two separate IAM identities (a service role AND an EC2 instance profile)?

<details>
<summary>Answer</summary>

The **service role** (`elasticmapreduce.amazonaws.com` trust) is used by EMR to provision and manage the cluster infrastructure — launching EC2 instances, configuring security groups, managing network interfaces. 

The **EC2 instance profile** (`ec2.amazonaws.com` trust) is assigned to the worker nodes themselves — this governs what the actual Spark/Hadoop processes running on those nodes can do (read/write S3, write logs to CloudWatch). 

They have different trust principals and different permission sets. Pre-deprecated roles like `AmazonEMREC2InstanceProfile` tried to combine them, which caused permission issues.

</details>

---

### 1.4 Inspect the Redshift Role

1. Click `trainee01-<BATCH_ID>-redshift-s3-role`
2. Open the inline policy JSON
3. Find the `RedshiftSpectrum` statement

**Question 5:** What Glue actions are allowed in the Redshift role's Spectrum statement? Why does Redshift need Glue permissions?

<details>
<summary>Answer</summary>

`glue:GetDatabase`, `glue:GetTable`, `glue:GetPartitions` — Redshift Spectrum queries S3 data using the Glue Data Catalog as its table/schema registry. When you run `SELECT * FROM external_schema.table`, Redshift calls Glue to look up where the data lives, what schema it has, and what partitions exist.

</details>

---

## Part 2: CLI — Query IAM Roles Programmatically

In your terminal (with `AWS_PROFILE=aws-de-lab` set):

### 2.1 List your roles

```bash
aws iam list-roles --query "Roles[?contains(RoleName, 'trainee01')].[RoleName, Arn]" --output table
```

Expected output:
```
--------------------------------------------------------------------------------------------------------------
|                                               ListRoles                                                    |
+-------------------------------------------+-----------------------------------------------------------------+
|  trainee01-<BATCH_ID>-emr-ec2-role           |  arn:aws:iam::<ACCOUNT_ID>:role/trainee01-<BATCH_ID>-emr-ec2-role  |
|  trainee01-<BATCH_ID>-emr-service-role       |  ...                                                            |
|  trainee01-<BATCH_ID>-firehose-role          |  ...                                                            |
|  trainee01-<BATCH_ID>-glue-service-role      |  ...                                                            |
|  trainee01-<BATCH_ID>-lakeformation-role     |  ...                                                            |
|  trainee01-<BATCH_ID>-redshift-s3-role       |  ...                                                            |
+-------------------------------------------+-----------------------------------------------------------------+
```

### 2.2 Get a specific role

```bash
aws iam get-role --role-name trainee01-<BATCH_ID>-glue-service-role
```

Examine the `AssumeRolePolicyDocument` field — this is the trust policy.

### 2.3 List inline policies on a role

```bash
aws iam list-role-policies --role-name trainee01-<BATCH_ID>-glue-service-role
```

### 2.4 Read an inline policy document

```bash
aws iam get-role-policy \
  --role-name trainee01-<BATCH_ID>-glue-service-role \
  --policy-name glue-inline-policy
```

This returns the permission policy JSON. Count the `Statement` array entries — you should see entries for:
- `GlueS3ObjectAccess`
- `GlueS3ListBuckets`
- `GlueServiceActions`
- `GlueLogs`
- `AllowPassRole`

### 2.5 List managed policy attachments

```bash
aws iam list-attached-role-policies --role-name trainee01-<BATCH_ID>-glue-service-role
```

---

## Part 3: Verify Your Own Permissions (Boundary Testing)

Test the principle of least privilege by verifying what you can and cannot do.

### 3.1 Things you CAN do

```bash
# List S3 buckets with your prefix (empty today — storage provisioned Day 16)
aws s3 ls | grep trainee01

# Get your own caller identity
aws sts get-caller-identity

# List Kinesis streams
aws kinesis list-streams

# Describe Redshift clusters
aws redshift describe-clusters --query 'Clusters[?contains(ClusterIdentifier, `trainee01`)]'
```

### 3.2 Things you CANNOT do (expected failures — educational)

Try each and observe the error:

```bash
# Attempt to create an IAM role (SHOULD FAIL)
aws iam create-role \
  --role-name test-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
```

Expected error:
```
An error occurred (AccessDenied) when calling the CreateRole operation: 
User: arn:aws:iam::<ACCOUNT_ID>:user/trainee01 is not authorized to perform: 
iam:CreateRole because no identity-based policy allows the iam:CreateRole action
```

```bash
# Attempt to view another attendee's resources (SHOULD FAIL)
aws s3 ls s3://trainee02-<BATCH_ID>-raw/

# Attempt to create a budget (SHOULD FAIL)
aws budgets create-budget \
  --account-id <ACCOUNT_ID> \
  --budget '{"BudgetName":"test","BudgetLimit":{"Amount":"1","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}'
```

**Question 6:** Why is it important that `iam:Create*` is in an explicit **Deny** statement rather than simply being omitted from the Allow statements?

<details>
<summary>Answer</summary>

In AWS IAM, **Deny always overrides Allow**. If the policy simply omitted `iam:Create*`, a future administrator might add a managed policy (like `AdministratorAccess`) to your user that grants it. An explicit Deny using the `Deny` effect cannot be overridden by any other `Allow` policy (except a root session). This "belt and suspenders" approach is critical in shared accounts.

</details>

---

## Part 4: Understand Policy Evaluation Logic

Work through this policy evaluation exercise mentally:

### Scenario

You are `trainee01`. You run:

```bash
aws s3 ls s3://trainee01-<BATCH_ID>-raw/
```

**Policy evaluation steps (answer each):**

1. Is there an explicit **Deny** matching `s3:ListBucket` on `trainee01-<BATCH_ID>-raw`? → `DenyIAMWriteOrgAccount` only denies IAM actions, so: **No**
2. Is there an explicit **Allow**? → Check `S3OwnBuckets` statement: `"arn:aws:s3:::trainee01-*"` matches → **Yes**
3. Result: **Allow**

Now try:

```bash
aws s3 ls s3://trainee02-<BATCH_ID>-raw/
```

1. Explicit Deny? No IAM deny covers S3 on other-attendee buckets directly
2. Explicit Allow? `S3OwnBuckets` is `trainee01-*` — `trainee02-<BATCH_ID>-raw` does **not** match
3. Result: **Implicit Deny** (no Allow → default deny)

**Question 7:** What is the difference between an **explicit deny** and an **implicit deny** in IAM?

<details>
<summary>Answer</summary>

- **Implicit Deny**: No policy grants access — the default position in AWS IAM. Everything is denied unless explicitly allowed.
- **Explicit Deny**: A `"Effect": "Deny"` statement actively prohibits an action. This overrides any `Allow` statement, even from other policies (e.g. a managed policy), except those in SCP allow lists. Explicit Deny is used for hard guardrails that must never be overridden.

</details>

---

## Part 5: Console Verification — Instance Profiles

1. In IAM console → **Instance profiles** (left sidebar, scroll down under "Access management")
2. Search `trainee01`
3. Click `trainee01-<BATCH_ID>-emr-instance-profile`
4. Note the **Role** listed — should be `trainee01-<BATCH_ID>-emr-ec2-role`

**Why the instance profile wrapper?**  
EC2 instances cannot directly assume IAM roles. Amazon EC2 uses the **instance profile** as the mechanism to inject role credentials into an EC2 instance's metadata endpoint (`http://169.254.169.254/latest/meta-data/iam/security-credentials/`). EMR worker nodes retrieve their S3/CloudWatch credentials from this endpoint automatically.

---

## Wrap-Up Checklist

Before leaving this lab, confirm you can:

- [ ] List all 6 IAM roles created for your prefix in the console
- [ ] Read a trust policy and explain what it means
- [ ] Read an inline permission policy JSON in the console
- [ ] Use `aws iam get-role-policy` to retrieve an inline policy via CLI
- [ ] Explain why `iam:PassRole` is required
- [ ] Explain the difference between EMR service role and EMR EC2 instance profile
- [ ] Demonstrate an expected `AccessDenied` on `iam:CreateRole`
- [ ] Explain explicit deny vs implicit deny

---

## Reflection Questions

1. If you were setting up your own AWS account for a data platform, how would you decide how many IAM roles to create vs. using a single catch-all role?

2. The Glue role allows `glue:*` on `Resource: "*"`. Is this a violation of least privilege? Why or why not? *(Hint: check whether Glue API actions support resource-level permissions)*

3. After Day 16, the instructor updates the IAM inline policies with real S3 bucket ARNs. What specific changes would you expect in the policy JSON? Which roles are affected?

---

*Back to: [Demo — IAM Role Creation](demo-iam-role-creation.md)*  
*Next: [Demo — Billing and Budget Alerts](demo-billing-budget-alerts.md)*  
*Tomorrow: [Day 16 — Storage with S3 and KMS](../day16/)*
