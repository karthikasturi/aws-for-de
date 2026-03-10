# Day 16 Lab: S3 Core Operations — Objects, Versioning, Lifecycle, and Security

**Type:** Attendee Self-Service Lab  
**Duration:** ~90 minutes  
**Day:** 16 — AWS Object Storage  
**Prerequisites:** Instructor has pre-provisioned 7 S3 buckets per attendee (done before Day 16 starts)

---

## Overview

In this lab you will work entirely within your own pre-provisioned S3 buckets. You will practice the full object lifecycle — upload, version, archive, secure, and monitor — using both the AWS Console and the AWS CLI.

> **Your bucket namespace**  
> All your buckets follow the pattern: `traineeNN-<BATCH_ID>-<descriptor>`  
> Replace `traineeNN` with your actual attendee ID throughout this lab (e.g. `trainee05`).

---

## Your Pre-Provisioned Buckets

Before this lab starts, the instructor has created these 7 buckets for each attendee:

| Bucket name | Purpose | Versioning | Lifecycle |
|---|---|---|---|
| `traineeNN-<BATCH_ID>-raw` | Raw ingested data | ✅ Enabled | None |
| `traineeNN-<BATCH_ID>-transformed` | Processed/clean data | ✅ Enabled | None |
| `traineeNN-<BATCH_ID>-scripts` | Glue/EMR job scripts | ✅ Enabled | None |
| `traineeNN-<BATCH_ID>-logs` | Application logs | ❌ Off | Delete after **30 days** |
| `traineeNN-<BATCH_ID>-athena-output` | Athena query results | ❌ Off | Delete after **7 days** |
| `traineeNN-<BATCH_ID>-glue-temp` | Glue temp storage | ❌ Off | Delete after **3 days** |
| `traineeNN-<BATCH_ID>-firehose-landing` | Kinesis Firehose output | ❌ Off | None |

All buckets have:
- Public access **fully blocked** (all four ACL/policy settings)
- Server-side encryption with **SSE-KMS** (your batch KMS key)

---

## Part 1: Explore Pre-Provisioned Buckets in the Console

### 1.1 Navigate to S3

1. In the AWS Console search bar, type **S3** and click the service
2. In S3, the **Buckets** list shows all buckets in the account — scroll and notice that every bucket name starts with an attendee prefix

> **Key observation:** S3 namespace is account-wide. You can *see* other attendees' bucket names but you cannot access their contents. You will verify this in Part 5.

### 1.2 Inspect your raw bucket

1. Click `traineeNN-<BATCH_ID>-raw`
2. Click the **Properties** tab — scroll through:
   - **Bucket Versioning** → should show `Enabled`
   - **Default encryption** → should show `SSE-KMS` with your batch key alias
   - **Block Public Access** → all four settings should be `On`
3. Click the **Management** tab → **Lifecycle rules** → note: no lifecycle rule yet (you will add one in Part 3)
4. Go back and click the **Objects** tab
5. Find `datasets/taxi_trips_sample.csv` — your instructor uploaded this sample dataset for you

### 1.3 Inspect a bucket with a lifecycle rule

1. Click `traineeNN-<BATCH_ID>-logs`
2. Click the **Management** tab → **Lifecycle rules**
3. Click the `delete-after-30-days` rule — observe:
   - **Filter scope:** all objects (empty prefix = bucket-wide)
   - **Expiration:** 30 days after creation
   - **Transition rules:** none
4. Repeat for `traineeNN-<BATCH_ID>-athena-output` (7-day delete) and `traineeNN-<BATCH_ID>-glue-temp` (3-day delete)

> **Discussion point:** Why does the Athena output bucket expire objects after 7 days? Why does glue-temp expire after 3?

---

## Part 2: Object Operations — Upload, Download, and Delete

### 2.1 Create a small sample data file

On your local machine (or workstation), create this file as `trip_update.csv`:

```csv
vendor_id,pickup_datetime,dropoff_datetime,passenger_count,trip_distance,pickup_lon,pickup_lat,dropoff_lon,dropoff_lat,payment_type,fare_amount,total_amount
CMT,2024-02-01 09:00:00,2024-02-01 09:22:00,1,4.10,-73.9851,40.7589,-73.9500,40.7700,Credit,16.00,18.50
VTS,2024-02-01 10:15:00,2024-02-01 10:30:00,2,2.30,-74.0060,40.7128,-73.9934,40.7400,Cash,10.00,10.00
CMT,2024-02-01 12:45:00,2024-02-01 13:10:00,1,5.80,-73.9772,40.7527,-73.8900,40.6500,Credit,21.00,24.00
```

### 2.2 Upload via Console

1. In your `traineeNN-<BATCH_ID>-raw` bucket, click **Upload**
2. Click **Add files** → select `trip_update.csv`
3. Expand **Properties** (before clicking Upload):
   - **Storage class:** keep as `Standard`
   - **Server-side encryption:** note it inherits Bucket default (SSE-KMS)
4. Click **Upload**

### 2.3 Inspect object metadata

1. Click the uploaded `trip_update.csv` object
2. On the object detail page, note:
   - **Object URL** — it is a private URL (returns `AccessDenied` in browser without auth)
   - **Storage class:** Standard
   - **Server-side encryption:** aws:kms
   - **Entity tag (ETag):** MD5 of the encrypted content
   - **Key Management Service key ARN:** your batch KMS key

### 2.4 Download and verify

1. Click **Download** — your browser downloads the file
2. Open it locally and confirm the 3 rows of data are intact

### 2.5 Copy object to another prefix

1. Select `trip_update.csv` → click **Actions** → **Copy**
2. Set destination: `s3://traineeNN-<BATCH_ID>-raw/archive/trip_update.csv`
3. Click **Copy**
4. Browse into the `archive/` prefix — confirm the copy exists

### 2.6 Delete an object

1. Select `archive/trip_update.csv` → **Delete**
2. Type `permanently delete` in the confirmation box → **Delete objects**
3. Note: with versioning enabled, the object shows a **delete marker** — it is not truly gone!

---

## Part 3: Versioning Deep-Dive

### 3.1 Verify versioning is enabled

1. In your `raw` bucket → **Properties** → **Bucket Versioning** → confirm `Enabled`

### 3.2 Upload multiple versions of the same object

**Version 1** — you already uploaded `trip_update.csv` in Part 2.

**Version 2** — edit `trip_update.csv` locally, add one new row:
```csv
VTS,2024-02-02 08:30:00,2024-02-02 08:50:00,3,3.50,-73.9645,40.7614,-73.9200,40.7100,Credit,14.00,16.50
```

Upload it to the **same key**: `trip_update.csv` (do not rename it).

**Version 3** — edit again, change `fare_amount` of the first row to `99.99`:
```csv
CMT,2024-02-01 09:00:00,2024-02-01 09:22:00,1,4.10,-73.9851,40.7589,-73.9500,40.7700,Credit,99.99,102.49
```

Upload again to `trip_update.csv`.

### 3.3 View version history in the Console

1. In the bucket, click **Show versions** toggle (top-right of Objects panel)
2. You should now see **3 version entries** for `trip_update.csv`, each with a unique Version ID
3. The topmost entry (most recent) is the **current version**

### 3.4 Download a specific version

1. Click the row for the **oldest version** (lowest in the list)
2. Click **Download** — this downloads Version 1 (the original 3-row file)
3. Verify the fare_amount for the first CMT row is `16.00` (not `99.99`)

### 3.5 Restore a previous version

To "restore" an older version, you simply copy it on top of the current version:

```bash
# CLI method — get the version ID of the version you want to restore
aws s3api list-object-versions \
  --bucket traineeNN-<BATCH_ID>-raw \
  --prefix trip_update.csv \
  --query 'Versions[*].[VersionId,LastModified]' \
  --output table

# Copy the desired version back to the same key (this creates a new latest version)
aws s3api copy-object \
  --bucket traineeNN-<BATCH_ID>-raw \
  --copy-source "traineeNN-<BATCH_ID>-raw/trip_update.csv?versionId=<VERSION_ID>" \
  --key trip_update.csv
```

After the copy, toggle **Show versions** again — you should have 4 version entries.

### 3.6 Understand delete markers

1. Select `trip_update.csv` (current version, versioning shown) → Delete
2. Toggle **Show versions** — notice a **Delete marker** appears as the latest entry
3. The object appears "deleted" but all previous versions are preserved
4. To **undelete**: select the Delete marker entry → Delete it → the previous version resurfaces as current

---

## Part 4: Lifecycle Rules — Add a Glacier Transition

You will add a lifecycle rule to your `raw` bucket to transition objects older than 90 days to **S3 Glacier Flexible Retrieval** (cost-efficient archival storage).

### 4.1 Create the rule in the Console

1. In `traineeNN-<BATCH_ID>-raw` → **Management** → **Lifecycle rules** → **Create lifecycle rule**
2. Fill in:

| Field | Value |
|---|---|
| Rule name | `archive-old-data` |
| Rule scope | Apply to all objects in the bucket |
| Checkbox: acknowledge | ✅ Check it |

3. Under **Lifecycle rule actions**, check:
   - ✅ **Transition current versions of objects between storage classes**
   - ✅ **Transition previous versions of objects between storage classes**

4. For **Current version transitions**:
   - Storage class: `Glacier Flexible Retrieval`
   - Days after object creation: `90`

5. For **Previous version transitions**:
   - Storage class: `Glacier Flexible Retrieval`
   - Days after objects become noncurrent: `30`

6. Click **Create rule**

### 4.2 Verify the rule

1. Back on the **Management** tab → **Lifecycle rules**
2. Your `archive-old-data` rule should show status **Enabled**
3. Click into the rule and review the Transitions section

> **Cost note:** Glacier Flexible Retrieval costs ~$0.004/GB/month vs Standard ~$0.023/GB/month — a ~6× cost reduction for data that is only accessed rarely. Retrieval of Glacier objects can take 3–5 hours.

### 4.3 Compare storage class transitions

The S3 storage class hierarchy from most to least frequent access (and highest to lowest cost):

| Storage Class | Use case | Retrieval time |
|---|---|---|
| Standard | Frequent access (daily) | Milliseconds |
| Standard-IA | Infrequent (monthly) | Milliseconds |
| One Zone-IA | Non-critical infrequent | Milliseconds |
| Intelligent-Tiering | Unknown access pattern | Milliseconds or hours |
| Glacier Instant Retrieval | Rare, instant needed | Milliseconds |
| Glacier Flexible Retrieval | Rare, hours OK | 1 min – 12 hours |
| Glacier Deep Archive | Compliance / long-term | 12 – 48 hours |

---

## Part 5: Encryption Inspection

### 5.1 View encryption configuration in the Console

1. Go to `traineeNN-<BATCH_ID>-raw` → **Properties** → **Default encryption**
2. Note:
   - Encryption type: **SSE-KMS**
   - AWS KMS key: the ARN of your batch KMS key (`alias/aws-de-lab-<BATCH_ID>-traineeNN`)
   - **Bucket Key:** Enabled (reduces KMS API calls by generating a short-lived bucket-level key — reduces cost)

3. Repeat for `traineeNN-<BATCH_ID>-scripts` — same configuration

### 5.2 Inspect the KMS key via CLI

```bash
export AWS_PROFILE=aws-de-lab

# List KMS key aliases to find your batch key
aws kms list-aliases --query "Aliases[?contains(AliasName, 'aws-de-lab')]" --output table

# Describe the key
aws kms describe-key \
  --key-id "alias/aws-de-lab-<BATCH_ID>-traineeNN" \
  --query 'KeyMetadata.{KeyId:KeyId, Description:Description, KeyState:KeyState, KeyRotation:KeyRotationStatus}' \
  --output table
```

> Note: `KeyRotationStatus` should be `true` — automatic key material rotation is enabled. Each year, AWS KMS automatically creates new key material without changing the key ARN.

### 5.3 Try to disable the KMS key (expect failure)

```bash
aws kms disable-key --key-id "alias/aws-de-lab-<BATCH_ID>-traineeNN"
```

Expected response:
```
An error occurred (AccessDeniedException) when calling the DisableKey operation:
User: arn:aws:iam::<ACCOUNT_ID>:user/traineeNN is not authorized to perform: kms:DisableKey
```

Your attendee policy only grants `kms:ListKeys`, `kms:DescribeKey`, and `kms:ListAliases` — mutating KMS resources is correctly blocked.

---

## Part 6: Enable Server Access Logging

Server access logging records all requests made to your bucket. You will enable logging on your `raw` bucket with logs written to your `logs` bucket.

### 6.1 Enable via Console

1. Go to `traineeNN-<BATCH_ID>-raw` → **Properties**
2. Scroll to **Server access logging** → click **Edit**
3. Set:
   - **Server access logging:** Enable
   - **Target bucket:** `traineeNN-<BATCH_ID>-logs`
   - **Target prefix:** `s3-access-logs/raw/`
4. Click **Save changes**

### 6.2 Generate some traffic

1. Upload `trip_update.csv` again (overwrite the current version)
2. Download the object twice
3. Navigate around the bucket in the console

### 6.3 Check for log objects

> **Wait 5–15 minutes** — access logs are delivered on a best-effort basis, typically within 15 minutes.

After waiting:
1. Go to `traineeNN-<BATCH_ID>-logs`
2. Browse the `s3-access-logs/raw/` prefix
3. Download a log file — each line is one request:

```
traineeNN-<BATCH_ID>-raw [15/Feb/2024:08:30:01 +0000] 203.0.113.5 arn:aws:iam::<ACCOUNT_ID>:user/traineeNN REST.PUT.OBJECT trip_update.csv "PUT /trip_update.csv HTTP/1.1" 200 - - 1234 120 - "-" "aws-cli/2.x" - ...
```

---

## Part 7: Cross-Attendee Access Control Test

This part proves that S3 resource isolation works — your prefix-scoped IAM policy prevents you from accessing any other attendee's buckets.

### 7.1 Try to list another attendee's bucket (Console)

1. In the S3 console bucket list, click on `trainee01-<BATCH_ID>-raw` (use a different attendee's bucket)
2. Expected result: the console shows an error or empty Objects tab with "Access Denied"

### 7.2 Confirm AccessDenied via CLI

```bash
export AWS_PROFILE=aws-de-lab

# Your own bucket — should succeed
aws s3 ls s3://traineeNN-<BATCH_ID>-raw/

# Another attendee's bucket — should fail
aws s3 ls s3://trainee01-<BATCH_ID>-raw/
```

Expected output for the second command:
```
An error occurred (AccessDenied) when calling the ListObjectsV2 operation:
Access Denied
```

### 7.3 Why does this work?

Walk through the IAM policy evaluation for `s3:ListBucket` on `trainee01-<BATCH_ID>-raw`:

1. **Explicit Deny?** → The `DenyIAMWriteOrgAccount` statement denies IAM actions only. No S3 deny exists.
2. **Explicit Allow?** → The `S3OwnBuckets` statement allows `s3:*` on `arn:aws:s3:::traineeNN-*`. The bucket `trainee01-<BATCH_ID>-raw` does NOT match your prefix (`traineeNN-*`).
3. **Default implicit deny** → No Allow matches → **AccessDenied**. ✅

### 7.4 Presigned URLs — Temporary Object Sharing

A presigned URL embeds credentials that allow temporary, unauthenticated access to a specific private object.

```bash
# Generate a presigned URL valid for 5 minutes (300 seconds)
aws s3 presign s3://traineeNN-<BATCH_ID>-raw/trip_update.csv \
  --expires-in 300

# Output will look like:
# https://traineeNN-<BATCH_ID>-raw.s3.ap-south-1.amazonaws.com/trip_update.csv?X-Amz-Algorithm=AWS4-HMAC-SHA256&...
```

1. Copy the output URL and open it in a browser **incognito window** (no AWS session)
2. The file downloads — even without AWS credentials, because the credentials are embedded in the URL
3. Wait 5 minutes and try again — the URL expires and returns `AccessDenied`

> **Security note:** Anyone who has the presigned URL can access the object until it expires. Never generate presigned URLs with very long expiry times for sensitive data.

---

## Part 8: CLI S3 Operations Summary

Quick reference for the most common S3 CLI patterns you will use throughout the course:

```bash
export AWS_PROFILE=aws-de-lab
export BUCKET="traineeNN-<BATCH_ID>-raw"

# List all your buckets
aws s3 ls | grep traineeNN

# List objects in a bucket
aws s3 ls s3://$BUCKET/

# List recursively (all prefixes)
aws s3 ls s3://$BUCKET/ --recursive --human-readable

# Upload a file
aws s3 cp trip_update.csv s3://$BUCKET/datasets/trip_update.csv

# Download a file
aws s3 cp s3://$BUCKET/datasets/trip_update.csv ./downloaded.csv

# Sync a local directory to S3 (only uploads changed files)
aws s3 sync ./local-folder/ s3://$BUCKET/folder/

# Copy between buckets (within your prefix)
aws s3 cp s3://$BUCKET/datasets/trip_update.csv \
  s3://traineeNN-<BATCH_ID>-transformed/datasets/trip_update.csv

# Delete an object
aws s3 rm s3://$BUCKET/datasets/trip_update.csv

# Inspect object metadata (head = no download)
aws s3api head-object \
  --bucket $BUCKET \
  --key datasets/taxi_trips_sample.csv

# Get bucket encryption configuration
aws s3api get-bucket-encryption --bucket $BUCKET

# Get bucket lifecycle configuration
aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET

# Get bucket versioning status
aws s3api get-bucket-versioning --bucket $BUCKET
```

---

## Lab Checkpoints

Before finishing, verify:

- [ ] You can list and navigate all 7 of your buckets in the console
- [ ] You uploaded `trip_update.csv` and can see 3+ versions in the version history
- [ ] You successfully restored an older version via CLI `copy-object`
- [ ] The `archive-old-data` lifecycle rule is active on your raw bucket
- [ ] You confirmed your KMS key is configured and rotation is enabled
- [ ] Server access logging is enabled on your raw bucket
- [ ] `aws s3 ls s3://trainee01-<BATCH_ID>-raw/` returns **AccessDenied**
- [ ] You generated a working presigned URL and it expired after 5 minutes

---

## Key Concepts Summary

| Concept | What you saw in this lab |
|---|---|
| **S3 bucket** | Logical container scoped to a region; globally unique name |
| **Versioning** | Every overwrite/delete creates a new version — data is never truly lost |
| **Delete marker** | Soft-delete in a versioned bucket; previous versions still retrievable |
| **Lifecycle rule** | Automates transition/expiration of objects based on age |
| **SSE-KMS** | Server-side encryption using AWS KMS — you own the key, AWS manages encryption |
| **Bucket Key** | Reduces KMS calls by caching a short-lived bucket-level data key |
| **Access logging** | Per-request audit log delivered to a target bucket |
| **Presigned URL** | Time-limited shareable link with embedded credentials |
| **Prefix isolation** | IAM policy restricts access to `traineeNN-*` ARNs; all else is implicitly denied |

---

## What's Next

In the [instructor demo](demo-s3-advanced-features.md), you will watch the instructor walk through features that require elevated permissions or external services:
- **CloudFront** — global content delivery network in front of S3
- **S3 Cross-Region Replication** — automatic replication requiring IAM role creation
- **CloudTrail + S3 integration** — API-level audit trail
- **S3 Transfer Acceleration** and **Snowball** — data transfer at scale
