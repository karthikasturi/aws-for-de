# Day 19 Demo: AWS Lake Formation — Fine-Grained Access Control

**Type:** Instructor Demo  
**Duration:** ~45 minutes  
**Day:** 19 — Serverless ETL and Queries  
**Why demo-only:** Lake Formation administrative operations (`RegisterResource`, `GrantPermissions`, `CreateDataCellsFilter`) require the `lakeformation:*` permission granted to the `lakeformation_role`, not to individual attendee IAM users. Attendees have read-only Lake Formation permissions: `GetDataAccess`, `ListPermissions`, `GetEffectivePermissionsForPath`.

---

## Instructor Checklist

```bash
export AWS_PROFILE=aws-de-lab
aws sts get-caller-identity   # Must return Account: "<ACCOUNT_ID>"
```

Use `trainee01` resources for the live demo. Console: **AWS Lake Formation**.

---

## Part 1: What Problem Does Lake Formation Solve? (10 min)

### Before Lake Formation: IAM everything

```
Without Lake Formation:
  Each analyst role needs:         IAM policies needed:
  ┌─────────────────┐              ┌────────────────────────────────┐
  │ analyst_alice   │──────────────│ s3:GetObject on s3://raw/*     │
  │ analyst_bob     │──────────────│ s3:GetObject on s3://raw/*     │
  │ analyst_finance │──────────────│ s3:GetObject on s3://raw/*.csv │
  └─────────────────┘              │ glue:GetTable on catalog/*     │
                                   │ glue:GetDatabase on *          │
  To revoke access to one table:   └────────────────────────────────┘
  → Edit every IAM policy
  To hide 3 columns from Bob:
  → Create a filtered view in Glue... or can't do it at all
```

### With Lake Formation: one governance plane

```
With Lake Formation:
  Lake Formation Admin
    │  Register S3 location (takes ownership)
    │  Grant permissions per table, per column, per row
    │
    ├─► analyst_alice: SELECT on taxi_trips (all columns)
    ├─► analyst_bob:   SELECT on taxi_trips EXCEPT pickup_lon, pickup_lat, dropoff_lon, dropoff_lat
    └─► analyst_finance: SELECT on taxi_trips WHERE payment_type = 'Credit'

  Athena, Redshift Spectrum, and Glue all respect Lake Formation permissions.
  One place to manage, audit, and revoke.
```

---

## Part 2: Lake Formation Setup in This Lab (10 min)

### Registered S3 locations

Lake Formation "owns" specific S3 paths. Any data access tool (Athena, Glue, Spectrum) must request access through Lake Formation's credential vending.

```bash
# Show registered locations
aws lakeformation list-resources \
  --query 'ResourceInfoList[*].{ResourceArn:ResourceArn,LastModified:LastModified}'
```

In this lab, two locations are registered:
- `s3://trainee01-2026-03-raw/` — raw CSV landing zone
- `s3://trainee01-2026-03-transformed/` — Parquet output

### Lake Formation administrators

```bash
# Show who the admins are
aws lakeformation get-data-lake-settings \
  --query 'DataLakeSettings.DataLakeAdmins'
```

Expected:
- `traineeNN-2026-03-lakeformation-role` — manages permissions for all catalog objects
- `traineeNN-2026-03-firehose-role` — Firehose needs write access to the registered location

### Current table permissions

```bash
# List permissions on the taxi_trips_parquet table
aws lakeformation list-permissions \
  --resource '{
    "Table": {
      "DatabaseName": "trainee01_2026_03_catalog",
      "Name": "taxi_trips_parquet"
    }
  }' \
  --region ap-south-1 \
  --query 'PrincipalResourcePermissions[*].{Principal:Principal.DataLakePrincipalIdentifier,Permissions:Permissions}'
```

---

## Part 3: Column-Level Security — the `no_location_data` Filter (10 min)

### What the column filter does

In the Terraform lab configuration, a `data_cells_filter` named `no_location_data` was created:

```hcl
# From modules/lake_formation/main.tf
# Note: create_column_filter = false by default — apply on-the-fly for the demo (see CLI step below)
resource "aws_lakeformation_data_cells_filter" "location_filter" {
  table_data {
    database_name    = "trainee01_2026_03_catalog"
    table_name       = "taxi"   # raw CSV table created by the Glue crawler
    name             = "no_location_data"
    column_names     = ["vendor_id", "pickup_datetime", "dropoff_datetime",
                        "passenger_count", "trip_distance",
                        "payment_type", "fare_amount", "total_amount"]
    # Excluded (PII-like GPS columns): pickup_lon, pickup_lat, dropoff_lon, dropoff_lat
    row_filter {
      all_rows_wildcard {}
    }
  }
}
```

This filter excludes the four GPS coordinate columns — useful if:
- Attendees should see trip data but not precise pickup/dropoff locations
- GDPR/privacy compliance: location = PII
- Data sharing with an external analytics partner

### Demo: Create and view the column filter

The filter is defined in Terraform (`create_column_filter = false` by default, so not pre-created). Create it live:

```bash
# Instructor: create the no_location_data column filter for the demo
aws lakeformation create-data-cells-filter \
  --table-data '{
    "TableCatalogId": "985539779502",
    "DatabaseName": "trainee01_2026_03_catalog",
    "TableName": "taxi",
    "Name": "no_location_data",
    "RowFilter": { "AllRowsWildcard": {} },
    "ColumnNames": ["vendor_id","pickup_datetime","dropoff_datetime","passenger_count","trip_distance","payment_type","fare_amount","total_amount"]
  }' --region ap-south-1
```

Then browse to it in the console:
1. Console → **Lake Formation** → **Data catalog** → **Tables**
2. Find `taxi` in `trainee01_2026_03_catalog`
3. Click **Data filters** tab
4. Show `no_location_data` filter:
   - **Row filter expression:** (none — all rows)
   - **Included column names:** all except the four GPS columns
   - **Excluded column names:** `pickup_lon`, `pickup_lat`, `dropoff_lon`, `dropoff_lat`

### Demo: Grant filtered access to a role

```bash
# Grant the filter to a specific analyst role (don't run — for demo)
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "arn:aws:iam::<ACCOUNT_ID>:role/analyst-role"}' \
  --resource '{
    "DataCellsFilter": {
      "TableCatalogId": "985539779502",
      "DatabaseName": "trainee01_2026_03_catalog",
      "TableName": "taxi",
      "Name": "no_location_data"
    }
  }' \
  --permissions '["SELECT"]' --region ap-south-1
```

```bash
# Instructor cleanup: delete the filter after Part 3
aws lakeformation delete-data-cells-filter \
  --table-catalog-id 985539779502 \
  --database-name trainee01_2026_03_catalog \
  --table-name taxi \
  --name no_location_data --region ap-south-1
```

When the analyst runs `SELECT *` in Athena, they will only see the allowed columns — the GPS columns will be absent from results.

---

## Part 4: Row-Level Security Demo (10 min)

Lake Formation also supports row-level filters using a SQL expression.

### Scenario: Payment type isolation

A business rule: the `finance` team can only query `Credit` payment records.

### Demo: Create row filter in console

1. Console → **Lake Formation** → **Data catalog** → **Tables** → `taxi_trips_parquet`
2. **Data filters** tab → **Create filter**
3. Fill in:
   - **Filter name:** `credit_payments_only`
   - **Row filter expression:** `payment_type = 'Credit'`
   - **Column access:** All columns
4. Show the preview pane (does not run against data, just validates syntax)

```bash
# Create via CLI (demo — run for trainee01 only)
aws lakeformation create-data-cells-filter \
  --table-data '{
    "TableCatalogId": "985539779502",
    "DatabaseName": "trainee01_2026_03_catalog",
    "TableName": "taxi_trips_parquet",
    "Name": "credit_payments_only",
    "RowFilter": {
      "FilterExpression": "payment_type = '\''Credit'\''"
    },
    "ColumnWildcard": {}
  }'
```

```bash
# Clean up after demo
aws lakeformation delete-data-cells-filter \
  --table-catalog-id 985539779502 \
  --database-name trainee01_2026_03_catalog \
  --table-name taxi_trips_parquet \
  --name credit_payments_only --region ap-south-1
```

---

## Part 5: LF-Tags — Attribute-Based Access Control (5 min)

Managing permissions table-by-table doesn't scale to thousands of tables. **LF-Tags** solve this.

### How LF-Tags work

```
1. Create a tag:      sensitivity = [public, internal, confidential, restricted]

2. Assign to resources:
   taxi_trips_parquet    → sensitivity = internal
   customer_pii          → sensitivity = restricted
   public_aggregates     → sensitivity = public

3. Grant by tag (not by table):
   analyst_role          → SELECT where sensitivity IN [public, internal]
   data_engineer_role    → SELECT, INSERT where sensitivity IN [public, internal, confidential]
   dpo_role              → SELECT where sensitivity = restricted

4. Add a new table with sensitivity = internal → analyst_role AUTOMATICALLY gets access
```

### Demo: Show LF-Tag catalog in console

1. Console → **Lake Formation** → **LF-tag management**
2. Click **Create LF-tag**:
   - **Key:** `sensitivity`
   - **Values:** `public`, `internal`, `confidential`, `restricted`
3. Cancel — just show the UI
4. Go to **Data permissions** → **LF-tag-based access control** tab
5. Explain: "Instead of granting on each table, you set it once at the tag level"

---

## Part 6: Audit Trail (5 min)

Lake Formation integrates with **CloudTrail** for a complete audit trail of data access.

```bash
# Show recent Lake Formation data access events in CloudTrail
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=lakeformation.amazonaws.com \
  --start-time $(date -d '-1 hour' --iso-8601=seconds) \
  --query 'Events[*].{Time:EventTime,User:Username,Event:EventName}' \
  --output table
```

Every `GetDataAccess` call — which Athena/Glue/Spectrum makes on behalf of the user — is logged with:
- Which IAM principal made the request
- Which S3 path (table location) was accessed
- Timestamp

This gives compliance teams a data access audit trail **separate** from S3 access logs.

---

## Architecture: How Athena Uses Lake Formation

```
1. Analyst runs SELECT in Athena
              │
              ▼
2. Athena calls Lake Formation GetDataAccess
              │
              ▼
3. Lake Formation:
   - Checks permissions for this principal + this table
   - Applies column/row filters if any
   - Issues time-limited, scoped S3 temporary credentials
              │
              ▼
4. Athena uses temp credentials to read S3 directly
   (only the allowed columns/rows pass through)
              │
              ▼
5. Results returned to analyst (filtered)
```

Without Lake Formation: Athena uses the Glue catalog directly and the IAM role's S3 permissions — no column or row filtering.

---

## Key Takeaways

| Feature | Lake Formation benefit |
|---|---|
| **Column security** | Hide sensitive columns (PII, GPS) without views or middleware |
| **Row security** | Restrict rows by business attribute (date range, payment type, region) |
| **LF-Tags** | Attribute-based access control — scales to thousands of tables |
| **Unified governance** | Athena + Glue + Redshift Spectrum + EMR all obey the same permissions |
| **Audit** | Every data access call logged to CloudTrail automatically |
| **Credential vending** | S3 credentials are scoped and time-limited per access request |

---

*Attendees: return to [Lab — Glue + Athena](lab-glue-athena.md)*  
*Tomorrow: [Day 20 — Kinesis Streaming Lab](../day20/lab-kinesis.md)*
