# Day 16 Demo: S3 Advanced Features — CloudFront, Replication, CloudTrail, and Data Transfer

**Type:** Instructor-Led Demo  
**Duration:** ~45 minutes  
**Day:** 16 — AWS Object Storage  
**Why demo only:** These features require IAM role creation, account-level permissions, or external hardware — all outside the attendee IAM policy scope.

---

## Overview

This demo covers advanced S3 capabilities that extend beyond the attendee sandbox permissions. You will watch the instructor configure each feature and explain the architecture. No attendee action is required.

> **Instructor prerequisite:** Verify account before starting.
> ```bash
> export AWS_PROFILE=aws-de-lab
> aws sts get-caller-identity   # Must return Account: "<ACCOUNT_ID>"
> ```

---

## Demo 1: Amazon CloudFront — CDN in Front of S3 (15 min)

### What it is

Amazon CloudFront is a global Content Delivery Network (CDN) with **450+ edge locations** (Points of Presence). When placed in front of an S3 bucket, CloudFront:
- Caches objects at the edge closest to the user
- Reduces latency for geographically distributed consumers
- Can serve private S3 content without making the bucket public (using Origin Access Control)
- Adds HTTPS, DDoS protection (AWS Shield Standard), and WAF integration

> **Why attendees can't do this:** The attendee IAM policy has no `cloudfront:*` actions.

### Demo steps

**Step 1 — Show the "static website" anti-pattern:**

```
In S3 Console → any bucket → Properties → Static website hosting
Point out: this requires making the bucket public OR using pre-signed URLs.
Neither is acceptable for production data.
CloudFront + Origin Access Control (OAC) is the correct pattern.
```

**Step 2 — Create a CloudFront distribution:**

1. Open **CloudFront Console** → **Create distribution**
2. Fill in:

| Field | Value |
|---|---|
| **Origin domain** | `trainee01-2026-03-raw.s3.ap-south-1.amazonaws.com` |
| **Origin access** | **Origin access control settings (recommended)** |
| **Origin access control** | Create new OAC (name: `demo-day16-oac`) |
| **Viewer protocol policy** | Redirect HTTP to HTTPS |
| **Allowed HTTP methods** | GET, HEAD |
| **Cache policy** | `CachingOptimized` (AWS managed) |
| **Price class** | Use only North America and Europe (cost demo only) |
| **Default root object** | `datasets/taxi_trips_sample.csv` |

3. Click **Create distribution**

**Step 3 — Update the S3 bucket policy:**

CloudFront will show a banner: *"The S3 bucket policy needs to be updated."*

Click **Copy policy** and show the JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::trainee01-2026-03-raw/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/EXAMPLEDIST"
        }
      }
    }
  ]
}
```

> **Key teaching point:** The bucket remains fully private. Only the specific CloudFront distribution identified by its ARN can read from it. This is called **Origin Access Control (OAC)** — the modern replacement for Origin Access Identity (OAI).

Paste the policy into the S3 bucket → **Permissions** → **Bucket policy** → **Edit** → Save.

**Step 4 — Test the distribution:**

Wait ~2 minutes for the distribution to deploy (Status: `Enabled`). Then:

```bash
# Access via CloudFront domain (no AWS credentials needed — cached at edge)
curl -I https://EXAMPLEABC.cloudfront.net/datasets/taxi_trips_sample.csv

# Shows: X-Cache: Miss from cloudfront (first request, not cached)
# Second request: X-Cache: Hit from cloudfront
```

**Step 5 — Invalidation (cache busting):**

1. In CloudFront → distribution → **Invalidations** tab → **Create invalidation**
2. Object paths: `/*` (invalidate all)
3. Explain: each invalidation path costs $0.005 after the first 1,000 paths/month

### Architecture summary

```
User (anywhere)
    ↓
CloudFront Edge Location (450+ worldwide)
    ↓ Origin fetch (only on cache miss)
S3 Bucket (ap-south-1) — still fully private
    ↑
  OAC: only CloudFront can read
```

---

## Demo 2: S3 Cross-Region Replication (CRR) (10 min)

### What it is

Cross-Region Replication (CRR) automatically copies every new object written in a source bucket to a destination bucket in a **different AWS region**. Use cases:
- Compliance (data residency in multiple regions)
- Disaster recovery (RPO ≈ minutes)
- Latency reduction (replica closer to consumers)

### Why attendees can't do this

CRR requires:
1. Creating an **IAM role** with `s3:ReplicateObject` permissions → `iam:CreateRole` is **Denied** in the attendee policy
2. The destination bucket must exist in a different region → attendees are constrained to `ap-south-1`

### Demo steps

**Step 1 — Show the replication rule setup:**

1. Go to source bucket → **Management** → **Replication rules** → **Create replication rule**
2. Walk through the form, highlighting each field:

| Field | What it configures |
|---|---|
| Replication rule name | Identifier for the rule |
| Status | Enabled / Disabled |
| Source scope | Entire bucket OR objects matching a prefix/tag |
| Destination | Bucket ARN in another region (e.g. `us-east-1`) |
| IAM role | The role S3 will assume to PUT objects into the destination |
| Storage class | Can override to a cheaper class in destination |
| Replication Time Control (RTC) | 99.99% of objects replicated within 15 minutes — with SLA |

3. Stop before creating. Point out: **"AWS will offer to create the IAM role for you."** Click the auto-create button — show the trust policy + permissions that get created.

**Step 2 — Explain what replicates vs what doesn't:**

| Replicates ✅ | Does NOT replicate ❌ |
|---|---|
| New objects (after rule enabled) | Existing objects already in bucket |
| Object tags and metadata | Delete markers (by default) |
| Encrypted objects (with matching KMS permissions) | Objects in Glacier (need to restore first) |
| Objects created by other bucket owners | Lifecycle-transitioned objects |

> Point out: existing objects require a separate **S3 Batch Operations** job to backfill.

---

## Demo 3: AWS CloudTrail + S3 API Logging (5 min)

### What it is

AWS CloudTrail records **every API call** made to AWS services at the management event level. S3 **data events** (GetObject, PutObject, DeleteObject) are an opt-in CloudTrail event type.

Difference from S3 Access Logs:

| Feature | S3 Server Access Logs | CloudTrail Data Events |
|---|---|---|
| **Granularity** | Per HTTP request (including partial reads) | Per S3 API call |
| **Delivery** | Best effort, within ~15 min | Near real-time |
| **Format** | Space-delimited log lines | JSON (structured) |
| **Query** | Athena or grep | Athena, CloudWatch Insights, or EventBridge rules |
| **Cost** | Free (storage of logs only) | $0.10 per 100,000 data events |
| **Coverage** | Single bucket | Any/all buckets in the account |

### Demo steps

1. Open **CloudTrail** → **Trails** → **Create trail**
2. Show the **Data events** section:
   - Select **S3**
   - Choose: **All current and future S3 buckets** OR specific bucket
   - Event type: `Read` and/or `Write`
3. Destination: **Create new S3 bucket** (for trail logs)
4. **Do not create** — explain cost implications and show the pricing link

**One practical example — finding who deleted an object:**

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteObject \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --output json | python3 -m json.tool | head -60
```

---

## Demo 4: S3 Transfer Acceleration (5 min)

### What it is

Transfer Acceleration uses **CloudFront edge locations** to accelerate uploads to S3. Instead of uploading directly to the `ap-south-1` S3 endpoint, your client sends data to the nearest CloudFront edge, which then routes it to S3 over AWS's optimized backbone network.

Typical benefit: 50–500% faster uploads for large files from distant regions.

### Demo steps

1. Go to any S3 bucket → **Properties** → **Transfer acceleration** → **Edit**
2. Enable it — S3 generates an accelerated endpoint:
   ```
   trainee01-2026-03-raw.s3-accelerate.amazonaws.com
   ```
3. Show the [S3 Transfer Acceleration Speed Comparison Tool](https://s3-accelerate-speedtest.s3-accelerate.amazonaws.com/en/accelerate-speed-comparsion.html) — compares direct vs accelerated upload from your browser's location

**Cost note:**
- Regular upload to S3 in same region: $0.00/GB transfer (free)
- Regular upload cross-region: ~$0.08/GB
- Transfer Acceleration: **+$0.04/GB** on top of standard transfer costs
- Only worth it for cross-region or intercontinental uploads of large files

4. Disable after demo (no ongoing benefit in a same-region lab).

---

## Demo 5: AWS Snowball — Offline Data Transfer at Scale (5 min)

### What it is

AWS Snowball is a **physical appliance service** for transferring petabytes of data to/from AWS when:
- Network bandwidth is insufficient (would take weeks or months)
- Network connectivity is unreliable or unavailable
- Data sovereignty prevents internet transfer

### Device comparison

| Device | Storage | Network | Use case |
|---|---|---|---|
| **Snowball Edge Storage Optimized** | 80 TB usable | 10 GbE | Bulk data migration |
| **Snowball Edge Compute Optimized** | 28 TB usable | 25 GbE + EC2 | Edge compute + transfer |
| **AWS Snowmobile** | 100 PB | Truck | Datacenter migration |

### Workflow (console walkthrough only)

1. Open **AWS Snow Family** console
2. Show **Create job** form — walk through:
   - **Job type:** Import into S3 / Export from S3 / Local Compute and Storage
   - **Shipping address:** where device is delivered
   - **S3 bucket:** destination for imported data
   - **IAM role:** `SnowballImportExport` — what Snowball uses to write to your S3 bucket
   - **KMS key:** devices are encrypted with a KMS key you own
3. Show the usual turnaround: **device arrives 2–3 business days after order**
4. After shipping back: AWS ingests data into S3 within 5–7 business days

> **Key point for GCP attendees:** The GCP equivalent is **Transfer Appliance** (~1 PB per device). AWS Snowball is more granular — you can order just an 80 TB device for a single migration project.

---

## Demo 6: S3 Glacier Deep Archive — Vault and Compliance Features (5 min)

### S3 Glacier storage class vs S3 Glacier Vault

| | S3 Glacier storage classes | Amazon S3 Glacier Vault |
|---|---|---|
| **Accessed via** | S3 API (same as Standard) | Glacier API (separate) |
| **Managed in** | S3 lifecycle rules | Glacier console / API |
| **Object retrieval** | Flexible Retrieval: 1–12 hr; Deep Archive: 12–48 hr | Same tiers |
| **Compliance controls** | S3 Object Lock | Vault Lock (WORM policy — cannot modify/delete) |
| **Inventory** | S3 Inventory | Vault inventory (takes ~4 hours to generate) |

### Vault Lock demo

1. Open **S3 Glacier** console (separate from S3)
2. Show **Vault Lock** — once applied:
   - Objects cannot be deleted for the specified retention period
   - Even the root account cannot override it
   - Used for HIPAA, SEC, FINRA compliance

```
Show the Vault Lock policy JSON:
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyDeleteArchiveForTwoYears",
    "Effect": "Deny",
    "Principal": "*",
    "Action": ["glacier:DeleteArchive"],
    "Resource": "arn:aws:glacier:ap-south-1:<ACCOUNT_ID>:vaults/compliance-vault",
    "Condition": {
      "NumericLessThan": {
        "glacier:ArchiveAgeInDays": "730"
      }
    }
  }]
}
```

3. Contrast with **S3 Object Lock** (WORM mode available directly in S3 — simpler for most use cases):
   - Governance mode: admins can override
   - Compliance mode: nobody can delete before retention period — not even root

---

## Architecture Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│                         S3 Storage Architecture                      │
│                                                                       │
│  Internet Users                                                       │
│       │                                                               │
│       ▼                                                               │
│  ┌──────────────┐   cache hit    ┌───────────────────────────────┐  │
│  │  CloudFront  │ ─────────────► │  CloudFront Edge (450+ PoPs)  │  │
│  │  Distribution│                └───────────────────────────────┘  │
│  └──────────────┘   cache miss         │                             │
│                                        ▼ origin fetch                │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    S3 Bucket (ap-south-1)                     │   │
│  │  ┌──────────┐  ┌──────────┐  ┌─────────────────────────────┐ │  │
│  │  │ Standard │─►│ Standard │─►│ Glacier Flexible Retrieval  │ │  │
│  │  │ (active) │  │    IA    │  │ (archived after 90 days)    │ │  │
│  │  └──────────┘  └──────────┘  └─────────────────────────────┘ │  │
│  │                        ▲ lifecycle transitions                   │  │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Monitoring & Audit                                            │  │
│  │  S3 Access Logs ──► logs bucket                               │  │
│  │  CloudTrail Data Events ──► CloudTrail bucket                 │  │
│  │  S3 Storage Lens dashboards (account-level analytics)         │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

| Feature | When to use |
|---|---|
| **CloudFront + S3** | Public-facing static assets, global user base |
| **Cross-Region Replication** | DR, compliance, multi-region latency reduction |
| **CloudTrail Data Events** | Incident investigation, compliance auditing, anomaly detection |
| **Transfer Acceleration** | Large file uploads from distant regions over internet |
| **Snowball** | Migrations of 10 TB+ when bandwidth is a bottleneck |
| **S3 Glacier / Glacier Deep Archive** | Long-term retention at <0.5% of Standard storage cost |
| **S3 Object Lock / Vault Lock** | WORM compliance — data that must not be modified or deleted |

---

## Instructor Teardown Note

> These demo resources (CloudFront distribution, any test bucket policies modified) must be cleaned up manually after Day 16 — navigate to the CloudFront console and delete the distribution, and revert any bucket policy changes in the S3 console.
>
> ```bash
> # Disable and delete the CloudFront distribution (takes ~5 min to disable)
> aws cloudfront list-distributions --query 'DistributionList.Items[*].[Id,DomainName]' --output table
> aws cloudfront delete-distribution --id EXAMPLEDIST --if-match <ETag>
> ```

---

## What's Next

**Day 17** — Relational Databases on AWS (Amazon RDS)

The instructor will pre-provision a PostgreSQL `db.t3.micro` instance for each attendee before Day 17 starts.

Once the RDS instances are running, the instructor will also update the IAM inline policies for each attendee's Glue and EMR EC2 roles to replace the placeholder S3 ARNs with the real bucket ARNs from Day 16.
