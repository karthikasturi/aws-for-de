# Day 14 Demo: Billing and Cost Explorer

**Type:** Instructor Demo  
**Duration:** ~15 minutes  
**Audience:** Attendees observe while instructor screen-shares  
**Who runs this:** Instructor (requires billing-enabled AWS root/admin account access)

> **Why demo-only?**  
> Attendee IAM policies do not include `ce:*` (Cost Explorer) or `billing:*` access — this is intentional.  
> Billing data is account-wide and sensitive. The instructor demonstrates from an account with appropriate access.

---

## Instructor Checklist

Before starting, confirm you are logged in with an account that has:

- [ ] `ce:*` — Cost Explorer access
- [ ] `budgets:*` — AWS Budgets access
- [ ] `aws-portal:ViewBilling` — Billing console access

Verify via:
```bash
export AWS_PROFILE=aws-de-lab
aws sts get-caller-identity
# Confirm Account: "<ACCOUNT_ID>"
```

---

## Part 1: AWS Billing Dashboard

**Navigate:** Search → `Billing` → Billing and Cost Management

### What to show attendees

1. **Billing overview page**
   - Current month's estimated charges
   - Month-to-date breakdown by service
   - Point out the top cost drivers for the lab: EC2 (workstations), Redshift (ra3.large), RDS (db.t3.micro)

2. **Bills page** (left sidebar → Bills)
   - Drill into current month
   - Show **Service charges** breakdown — expand to see per-region line items
   - Explain: charges are accrued hourly; this is why all lab resources are torn down nightly

3. **Free Tier** (left sidebar → Free Tier)
   - Show which services are still within free tier limits
   - Explain: the shared sandbox account may have exhausted free tier on some services

### Key talking points

> "In GCP, you manage budgets and billing in the Cloud Console under Billing. AWS has an equivalent dedicated **Billing and Cost Management** console. AWS bills by the second for compute services (EC2, EMR, Redshift). Redshift charges an hourly *rate* but bills in one-second increments for partial hours — so a 5-minute Redshift cluster creation attempt still incurs a small charge."

| AWS | GCP Equivalent |
|---|---|
| Billing and Cost Management | Cloud Billing |
| Cost Explorer | Cloud Billing reports |
| AWS Budgets | Budget alerts in Cloud Billing |
| Cost Allocation Tags | Labels-based cost attribution |
| Savings Plans | Committed Use Discounts |
| Reserved Instances | 1-yr / 3-yr CUDs |

---

## Part 2: Cost Explorer

**Navigate:** Billing → Cost Explorer (or search → `Cost Explorer`)

> **Note:** Cost Explorer must be enabled once per account. If not yet enabled, click **Enable Cost Explorer** and note it takes up to 24 hours to populate data.

### What to show

1. **Default view** — Last 6 months, grouped by service
   - Identify the top 3–5 cost drivers in the account
   - Show the date range selector — switch to **14 day** view for granular daily spikes

2. **Group by dimension** — Change "Group by" to:
   - `Service` (default) — see S3 vs EC2 vs Redshift
   - `Tag: Batch` — confirm the `<BATCH_ID>` tag groups all lab resources
   - `Tag: Environment` — confirm `training` tag is applied everywhere

3. **Filter by tag** — Filter to `Environment = training`
   - This shows ONLY lab costs, excluding any non-lab resources

4. **Hourly granularity** — Switch to Today, Hourly view
   - Show the spend spike from when lab resources were provisioned on Day 14/15

### Key talking points

> "Cost Explorer uses the tags applied to every resource. Every resource has `Environment=training`, `Batch=<BATCH_ID>`, and `ManagedBy=lab`. This is why tagging discipline is not optional — it directly enables cost attribution and chargeback."

---

## Part 3: AWS Budgets

**Navigate:** Billing → Budgets → Create budget

### Step-by-step demo (create a budget, then cancel)

**Step 1:** Click **Create budget**

**Step 2:** Select **Use a template (simplified)** → choose **Monthly cost budget**

**Step 3:** Fill in:
| Field | Demo value |
|---|---|
| Budget name | `aws-de-lab-monthly-budget` |
| Budgeted amount ($) | `200` |
| Email recipients | instructor email |

**Step 4:** Click **Create budget** — OR explain the fields and then **Cancel** to avoid creating a real alert.

**Step 5:** Show the **Budgets dashboard** — show forecasted vs actual.

### Alert thresholds (explain)

When creating a budget, you can set multiple alert thresholds:

| Threshold | Trigger type | Example use |
|---|---|---|
| 80% of budget | Actual | "Warning: getting close" |
| 100% of budget | Actual | "Budget hit" |
| 110% of budget | Forecasted | "On track to exceed" |

### Key talking points

> "In GCP, you set budget alerts in Cloud Billing under Budgets & alerts. The AWS equivalent is AWS Budgets under the Billing console. One key difference: AWS Budgets can also trigger automated actions — for example, automatically applying an SCP to deny new resource creation when the budget is hit."

---

## Part 4: Cost Allocation Tags

**Navigate:** Billing → Cost allocation tags

### What to show

1. **User-defined tags** tab
   - Show tags: `Environment`, `Batch`, `ManagedBy`
   - Status should be **Active** (tags must be activated here before they appear in Cost Explorer)

2. Explain that these three tags are applied to every resource in the lab — either during creation in the console (via the Tags section) or enforced by IAM policy conditions.

   | Tag key | Value | Purpose |
   |---|---|---|
   | `Environment` | `training` | Isolates lab costs from any other usage in the account |
   | `Batch` | `<BATCH_ID>` | Groups all resources belonging to this training cohort |
   | `ManagedBy` | `lab` | Indicates lab-managed resource |
3. Without activating tags here in the Billing console, they don't appear in Cost Explorer reports. This is an AWS quirk — tag activation is a one-time billing console step.

---

## Part 5: Key Cost Controls in This Lab

Remind attendees of these cost controls that are built into the lab's architecture:

| Guard | Configuration | Why |
|---|---| ---|
| Glue job timeout | 30-minute maximum timeout on every Glue job | AWS default is 2880 min — could cost $100s |
| EMR auto-termination | idle timeout of 7200 seconds (2 hrs) | Auto-terminates idle clusters after 2 hrs |
| Amazon Data Firehose buffering | 60-second buffering interval | Minimises per-PUT charges |
| No NAT Gateway | Architecture decision | NAT costs ~$32/mo per AZ + data transfer |
| Nightly teardown | Instructor manually tears down day-specific resources each evening | Avoids overnight idle charges |

---

## Wrap-Up Talking Points

- AWS bills are complex — Cost Explorer and tags are your best tools for visibility
- Tagging discipline in IaC is directly tied to financial governance
- Always set a budget alert before provisioning long-running resources in a new account
- In this lab we tear down every night — but in production, rightsizing (Reserved Instances, Savings Plans) is critical

---

*Back to: [Lab — Console Exploration](lab-console-exploration.md)*  
*Next: [Day 15 — IAM Roles and Policies](../day15/demo-iam-role-creation.md)*
