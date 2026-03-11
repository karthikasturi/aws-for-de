# Day 17 Demo: RDS Advanced — Multi-AZ, Read Replicas, and Aurora

**Type:** Instructor Demo  
**Duration:** ~60 minutes  
**Day:** 17 — Databases on AWS  
**Why demo-only:** Multi-AZ and Read Replicas double/triple the instance cost. Creating Aurora clusters requires `rds:CreateDBCluster` at the account level, which is beyond the attendee prefix-scoped policy. Performance Insights is not enabled on `db.t3.micro`.

---

## Instructor Checklist

```bash
export AWS_PROFILE=aws-de-lab
aws sts get-caller-identity   # Must return Account: "<ACCOUNT_ID>"
```

Navigate to the **RDS Console** — use the lab's `trainee01-2026-03-postgres` instance for live demos.

---

## Part 1: RDS Engine Overview (10 min)

### Supported engines on Amazon RDS

| Engine | AWS-managed | Notes |
|---|---|---|
| **PostgreSQL** | ✅ | This lab — v15.x |
| **MySQL** | ✅ | Most popular web app DB |
| **MariaDB** | ✅ | MySQL-compatible fork |
| **Oracle** | ✅ | BYOL or License Included |
| **SQL Server** | ✅ | BYOL or License Included |
| **Amazon Aurora (MySQL)** | ✅ | AWS-native, up to 5× faster |
| **Amazon Aurora (PostgreSQL)** | ✅ | AWS-native, up to 3× faster |

### RDS vs Amazon Aurora

| | Amazon RDS | Amazon Aurora |
|---|---|---|
| **Storage** | EBS-backed (gp2/gp3, io1) | Distributed, auto-grows in 10 GiB chunks to 128 TiB |
| **Multi-AZ** | Standby replica (synchronous) | 6 copies across 3 AZs always |
| **Read Replicas** | Up to 5 | Up to 15 |
| **Failover time** | ~60–120 seconds | ~30 seconds (quorum-based) |
| **Backtrack** | No | Yes (MySQL-compatible) — rewind DB to a point in time without restore |
| **Serverless option** | No | Aurora Serverless v2 (auto-scaling ACUs) |
| **Cost** | Lower on small instances | Higher base cost, better at scale |

**When to use Aurora:**  
High-traffic OLTP, auto-scaling requirement, built-in HA with minimal ops overhead.

**When to use RDS:**  
Standard relational workloads, existing licenses (Oracle/SQL Server), tight cost budgets.

---

## Part 2: Multi-AZ Deployment Demo (15 min)

### What Multi-AZ does

```
  Primary Instance (AZ-a)          Standby Instance (AZ-b)
  ┌─────────────────────┐          ┌─────────────────────┐
  │   db.t3.micro        │  sync    │   db.t3.micro        │
  │   Writes → here      │ ────────►│   Receives changes   │
  │   Reads  → here      │          │   NOT readable       │
  └─────────────────────┘          └─────────────────────┘
         ↑                                  ↑
     Same DNS endpoint            (same endpoint — DNS flips on failover)
```

**Key properties:**
- Synchronous replication — every write is committed on standby before acknowledging to client
- **Same endpoint DNS** — your application connection string doesn't change on failover
- Standby is **not readable** (use Read Replicas for read scaling)
- Automatic failover triggers on: AZ outage, primary host failure, DB engine failure, maintenance

### Demo: Show Multi-AZ in console

1. Click `trainee01-2026-03-postgres` → **Configuration** tab
2. Show **Multi-AZ: No** — explain this is single-AZ for cost in the lab
3. Click **Modify** → scroll to **Availability and durability** → select **Multi-AZ DB instance**
4. Show the **cost warning** — it doubles the instance cost
5. **Do not apply** — click **Cancel**

### Demo: Failover scenario walkthrough

Explain the failover sequence on the whiteboard:

```
1. Primary host fails (hardware, AZ outage, OS crash)
       ↓
2. RDS detects failure (health check, ~10–30 seconds)
       ↓
3. DNS TTL expires for the endpoint (60–120 seconds)
       ↓
4. DNS switches to standby's IP
       ↓
5. Application reconnects (automatic if connection pool has retry logic)

Total application-visible downtime: ~60–120 seconds
```

**Best practice for applications:**
- Set JDBC/psycopg2 connection retries
- Use connection pooling (PgBouncer, RDS Proxy)
- Don't cache the resolved IP of the endpoint

---

## Part 3: Read Replicas (10 min)

### What Read Replicas do

```
Primary (read/write)
    │ asynchronous replication
    ├──────────────────► Read Replica 1 (same region or cross-region)
    ├──────────────────► Read Replica 2
    └──────────────────► Read Replica 3 (up to 5 for RDS, 15 for Aurora)
```

- **Asynchronous** — replicas may lag seconds to minutes behind primary
- **Readable** — direct SELECT queries, reports, analytics
- Can be promoted to become a standalone primary (disaster recovery)
- Can be in a different region (read locally, write centralised)

### Demo: Create a Read Replica (show steps, don't complete)

1. Console → `trainee01-2026-03-postgres` → **Actions** → **Create read replica**
2. Walk through the form:
   - **DB instance identifier:** `trainee01-2026-03-postgres-read-replica`
   - **Destination region:** same or different
   - **Instance class:** can be smaller (read replicas scale independently)
   - **Multi-AZ:** can add a standby to the replica
3. Point out: creating a replica briefly increases I/O on the source instance during the initial snapshot
4. **Cancel** — don't create it

### Read Replica vs Multi-AZ Summary

| | Multi-AZ Standby | Read Replica |
|---|---|---|
| **Purpose** | High availability | Read scaling |
| **Readable?** | ❌ No | ✅ Yes |
| **Lag** | Synchronous (zero) | Asynchronous (seconds to minutes) |
| **Failover** | Automatic | Manual promotion |
| **Same endpoint?** | ✅ Yes | ❌ Separate endpoint |
| **Cross-region?** | ❌ Same region only | ✅ Yes |

---

## Part 4: Performance Insights (10 min)

> Performance Insights is not available on `db.t3.micro` in this lab — not all instance classes support it. Demo uses a hypothetical larger instance or the demo account.

### What Performance Insights shows

**DB Load** — the average number of active sessions at any moment, broken down by:
- `wait/io/ aurora_redo_log_flush` — time waiting for I/O
- `CPU` — time being processed
- `wait/lock/table/sql/handler` — lock contention

### Demo steps (if a larger instance is available)

1. RDS Console → any `db.t3.small` or larger instance
2. **Monitoring** tab → scroll to **Performance Insights**
3. Show the **Top SQL** list — queries sorted by DB load contribution
4. Show the **Top waits** — I/O vs CPU vs lock
5. Toggle time range: 1 hour, 24 hours, 1 week
6. Point out: **Top hosts** and **Top users** dimensions help identify which application server or user is driving load

### Key metric: DB Load vs Max CPU

```
DB Load  =  average number of sessions actively executing
Max vCPUs = 2 (db.t3.small has 2 vCPUs)

If DB Load > Max vCPUs: you are CPU-bound → scale up or tune queries
If DB Load < Max vCPUs but high latency: likely I/O-bound → check storage IOPS
```

---

## Part 5: Automated Backups, Snapshots, and PITR (10 min)

### Backup types

| Type | Who creates it | Retention | Cost |
|---|---|---|---|
| **Automated backup** | RDS daily | 1–35 days (lab: 1 day) | Covered by free backup storage (= instance size) |
| **Manual snapshot** | You | Until you delete it | Charged per GB-month beyond free tier |
| **Export to S3** | You | Until you delete it | S3 storage + $0.01/GB export fee |

### Point-in-Time Recovery (PITR)

With automated backups enabled, you can restore a DB instance to **any second** within the retention window.

```
How PITR works:
1. RDS takes a daily automated backup snapshot
2. RDS continuously streams transaction logs to S3
3. To restore to a point in time:
   - RDS restores the most recent snapshot before your target time
   - Replays transaction logs up to the exact timestamp
   
Result: a NEW DB instance is created at the recovered point in time
        (your original instance is not affected)
```

### Demo: Show restore options

1. Console → `trainee01-2026-03-postgres` → **Actions** → **Restore to point in time**
2. Set **Custom date and time** — pick a time 30 minutes ago
3. Set a new identifier: `trainee01-2026-03-postgres-restored`
4. Walk through options (instance class, VPC, security group)
5. Point out: this creates a **new instance** — it takes 15–30 minutes
6. **Cancel** — don't proceed

---

## Part 6: RDS Proxy (5 min)

### Problem: Connection storms

A typical web application opens a new database connection for each request. Under high load:
- Thousands of simultaneous connections overwhelm the DB
- `db.t3.micro` supports ~85 max connections
- Each connection consumes memory on the DB server

### Solution: RDS Proxy

```
App Tier (thousands of connections)          RDS Proxy          RDS Instance
┌─────┐                                  ┌─────────────┐      ┌────────────┐
│ EC2 │──┐                               │             │      │            │
├─────┤  ├─── pool of "super-user"  ─────│  multiplexes│──────│  50 max    │
│ EC2 │──┤      connections              │  connections│      │  connections│
├─────┤  │                               │             │      │            │
│ EC2 │──┘                               └─────────────┘      └────────────┘
thousands of app connections                                   manageable load
```

**Key features:**
- Connection pooling — maintains a pool, multiplexes thousands of app connections into fewer DB connections
- Automatic failover — routes to standby during Multi-AZ failover, cutting downtime from ~60s to ~3–5s
- IAM authentication — supports IAM tokens instead of passwords
- Secrets Manager integration — automatically rotates credentials without restarting the app

**When to use:**
- Serverless (Lambda) applications — Lambda opens a new connection on every invocation
- Auto-scaling application tier with highly variable connection counts
- Any application that cannot handle connection interruptions gracefully

---

## Part 7: Cost Summary

Show cost comparison (see official pricing for ap-south-1):

| Configuration | Notes | Cost indicator |
|---|---|---|
| db.t3.micro single-AZ | Lab instance | Lowest |
| db.t3.micro Multi-AZ | 2× instance cost | Low |
| db.t3.micro + Read Replica | 2× instance cost | Low |
| db.t3.small single-AZ | Performance Insights available | Medium |
| db.r8g.large Multi-AZ | Production-grade | High |
| Aurora Serverless v2 | 0.5 to 128 ACUs, auto-scale | Variable |

> Always check the [Amazon RDS pricing page](https://aws.amazon.com/rds/postgresql/pricing/) for current ap-south-1 rates. Costs change and vary by instance generation.

---

## Key Takeaways

| Concept | Rule of thumb |
|---|---|
| **Multi-AZ** | Always on for production. Off for dev/test/labs |
| **Read Replicas** | Use for read-heavy analytics queries; remember asynchronous lag |
| **Snapshots** | Automated for RPO, manual for pre-change safety |
| **PITR** | Enabled as long as backup_retention_period > 0 |
| **RDS Proxy** | Essential for Lambda or rapidly scaling app tiers |
| **Aurora** | Step up from RDS when you need higher throughput, global tables, or serverless auto-scale |

---

*Attendees: return to [Lab — RDS PostgreSQL](lab-rds-postgresql.md)*  
*Tomorrow: [Day 18 — Redshift Data Warehousing](../day18/lab-redshift.md)*
