# Day 18 Demo: Redshift Advanced — WLM, Concurrency Scaling, and Serverless

**Type:** Instructor Demo  
**Duration:** ~45 minutes  
**Day:** 18 — Data Warehousing with Amazon Redshift  
**Why demo-only:** WLM configuration requires superuser / cluster admin access. Concurrency Scaling, Redshift Serverless, and RA3 managed storage details are account-level settings. Resize and elastic resize change billing mid-session.

---

## Instructor Checklist

```bash
export AWS_PROFILE=aws-de-lab
aws sts get-caller-identity   # Must return Account: "<ACCOUNT_ID>"
```

Use `trainee01-2026-03-redshift` for live demos in the console.

---

## Part 1: Redshift Architecture Recap (10 min)

### Cluster anatomy

```
                        ┌─────────────────────────────────────────┐
                        │          Amazon Redshift Cluster         │
                        │                                          │
  Client ──── JDBC ────►│  Leader Node                            │
                        │   - Receives queries                     │
                        │   - Builds query execution plan          │
                        │   - Aggregates results from computes     │
                        │                                          │
                        │  Compute Node 1        Compute Node N   │
                        │   - Node slice 0         - Node slice   │
                        │   - Node slice 1         - Node slice   │
                        │   - Executes plan        - Executes plan │
                        │   - Local columnar       - Local columnar│
                        │     storage (RA3 → S3)    storage       │
                        └─────────────────────────────────────────┘
```

**Node slices:** Each compute node is divided into slices. Each slice gets an independent portion of the node's memory and disk, processing its portion of data in parallel.

- `ra3.large`: 2 slices per node
- `ra3.xlplus`: 4 slices per node
- `ra3.4xlarge`: 12 slices per node

Our lab uses `ra3.large` single-node = 2 slices total.

### RA3 managed storage

Traditional Redshift (DS2/DC2 nodes) stored data **on local SSDs** — you paid for compute AND storage always together.

RA3 separates compute from storage:
- **Compute (vCPUs, RAM):** You scale independently
- **Managed storage:** Data lives in S3, cached in fast local NVMe on the node
- **You pay for:** Compute (per hour) + Managed storage (per GB-month in S3)

Demo in console: Click `trainee01-2026-03-redshift` → **Cluster performance** → show the **Managed storage** section.

---

## Part 2: Workload Management (WLM) (10 min)

### Problem: A 3-second query and a 3-hour query share the same queue

Without WLM, a heavy ETL job consuming all memory delays an analyst's quick lookup.

### WLM queues

Each Redshift cluster has a **query queue** system. Queries are routed to queues, and each queue has a memory allocation and concurrency limit.

**Automatic WLM (recommended — enabled by default):**

```
Automatic WLM adjusts memory and concurrency dynamically based on observed workload.
Short queries get a fast lane, long queries get full memory when they're alone.

Queue          Memory %    Concurrency    Timeout
auto-eval      dynamic     auto           none
```

**Manual WLM:**

```
Queue          Memory %    Concurrency    Timeout      User group
SLA_critical   40%         5              30s          sla_users
ETL_jobs       50%         2              3600s        etl_users
default        10%         5              none         all others
```

### Demo: View WLM configuration

1. Console → `trainee01-2026-03-redshift` → **Properties** tab → scroll to **Workload management**
2. Click the parameter group: `trainee01-2026-03-rg`
3. Show the current WLM JSON config
4. Walk through what each field means: `query_groups`, `user_groups`, `memory_percent_to_use`, `max_execution_time`

### Demo: STL and STV system tables for monitoring

```sql
-- In Query Editor v2 or psql as labadmin

-- Currently running queries
SELECT query, pid, starttime, query_text
FROM stv_recents
WHERE status = 'Running';

-- Queue wait times
SELECT query, queue_time, exec_time
FROM svl_query_summary
ORDER BY starttime DESC
LIMIT 10;

-- Identify slow queries (> 10 seconds)
SELECT userid, query, starttime, endtime,
       DATEDIFF(second, starttime, endtime) AS duration_sec,
       SUBSTRING(querytxt, 1, 100)          AS query_snippet
FROM stl_query
WHERE DATEDIFF(second, starttime, endtime) > 10
ORDER BY starttime DESC
LIMIT 20;
```

---

## Part 3: Concurrency Scaling (5 min)

### Problem: 50 analysts hit the cluster simultaneously at month-end

Single-cluster concurrency limit: 50 concurrent queries on `ra3.large`.

**Concurrency Scaling** automatically adds read-only cluster capacity for burst workloads.

```
Normal load:    ─────────── Main cluster handles all ────────────
                             (within concurrency limit)

Burst load:
                ─────────── Main cluster at limit ───────────────
                         ╰─ Concurrency scaling cluster added ──╮
                                                                (ephemeral)
                ─────────── Back to main cluster ───────────────

Billing: First 1 hour/day is FREE, then $0.008/second per added cluster
```

### Demo: Enable Concurrency Scaling

1. Console → Parameter group → click **Edit** → find `wlm_json_configuration`
2. Show the `"concurrency_scaling": "auto"` option in the WLM JSON
3. Show where to set the **Concurrency Scaling mode** per queue: `off / auto`
4. Note: Concurrency Scaling only applies to **read queries** (SELECT)

---

## Part 4: Redshift Serverless (10 min)

### Traditional Redshift vs Redshift Serverless

| | Redshift Provisioned | Redshift Serverless |
|---|---|---|
| **Billing** | Per compute node, per hour (always on) | Per RPU-second when active (scale to zero) |
| **Scaling** | Manual resize or elastic resize | Automatic, instant RPU scaling |
| **Best for** | Consistent, predictable workloads | Sporadic/unpredictable workloads |
| **Cold start** | None (always running) | Near-zero (warm pool management) |
| **Concurrency Scaling** | Optional add-on | Built-in |
| **Min cost** | ~$0.40+/hr (ra3.large) | $0 when idle |

**RPU (Redshift Processing Units):** The unit of capacity for Serverless. Each RPU is vCPU + memory bundle. Configure base (minimum) RPUs and max RPUs.

### Demo: Show Serverless in console

1. Console → Left navigation → **Redshift Serverless**
2. Click **Try Amazon Redshift Serverless** (or show the existing setup)
3. Walk through: **Namespace** (logical container = DB + users + schemas) vs **Workgroup** (compute config)
4. Show: base RPU = 8 (minimum), max RPU = 512 (scales automatically)
5. Explain: data is stored in the same RA3 managed storage (S3-backed) — you can even query from a Serverless workgroup and a provisioned cluster using the same data

### When to recommend Serverless

- Variable/unpredictable query volume
- Dev/test environments (pay only when queries run)
- BI tools used by many users who query infrequently
- New projects where usage patterns are unknown

---

## Part 5: Resize Operations (5 min)

### Elastic Resize (fast, ~5–10 minutes)

Doubles or halves the number of nodes of the **same node type**:
- `ra3.large` 1-node → `ra3.large` 2-node
- During resize: cluster is **briefly unavailable** (~5 minutes), data is redistributed in background

### Classic Resize (slow, hours)

Change node type OR make a large scaling change:
- `ra3.large` → `ra3.xlplus` (different node type)
- Takes hours, cluster is in **read-only mode** during resize

### Demo: Show resize in console

1. Console → `trainee01-2026-03-redshift` → **Actions** → **Resize**
2. Show **Elastic resize** tab — select new node count
3. Show estimated time
4. Switch to **Classic resize** — show node type dropdown
5. **Cancel** — do not resize

---

## Part 6: Data Sharing (5 min)

Redshift Data Sharing allows clusters to share **live data** without copying it:

```
Producer Cluster          Consumer Clusters
┌──────────────┐          ┌──────────────┐
│              │   share  │   Read-only  │
│  sales_data  │ ────────►│   access to  │
│   (live)     │          │  sales_data  │
└──────────────┘          └──────────────┘
                          ┌──────────────┐
                          │   BI cluster │
                          │   (can also  │
                          │   read live) │
                          └──────────────┘
```

**Use cases:**
- Separate "write" cluster (ETL) from "read" cluster (BI tools)
- Share data across departments without ETL pipelines
- Cross-account sharing via AWS Data Exchange

**Demo: Show data share creation** (walkthrough, don't create):
1. Console → Cluster → **Data shares** tab
2. Click **Create data share**
3. Walk through: share name, objects to share (schemas/tables/views), consumer accounts
4. Explain: consumer sees the data with ~zero latency (same managed storage)

---

## Key Takeaways

| Concept | Recommendation |
|---|---|
| **DISTKEY** | Use for large fact tables, match to most common join column |
| **SORTKEY** | Use for time-series data (most queries filter on time) |
| **ENCODE/compression** | Let `ANALYZE COMPRESSION` suggest encodings for large tables |
| **VACUUM** | Run after large deletes/updates; automatic in background for most cases |
| **WLM** | Use Automatic WLM unless you have strict SLA requirements |
| **Concurrency Scaling** | Enable for variable workloads at no extra cost for the first hour/day |
| **Redshift Serverless** | Dev/test, unpredictable BI workloads |
| **Spectrum** | Archive cold data in S3 Parquet, query via Spectrum — save storage costs |

---

*Attendees: return to [Lab — Redshift](lab-redshift.md)*  
*Tomorrow: [Day 19 — Glue, Athena, and Lake Formation](../day19/lab-glue-athena.md)*
