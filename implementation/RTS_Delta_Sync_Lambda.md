# Implementation: RTS Delta Sync Lambda (EventBridge-Driven)

**Belongs to:** [TRS Design Document v4](../TRS_Design_Document_v4.md)  
**Component:** RTS Relationship Delta Sync — AWS Lambda  
**Trigger:** Amazon EventBridge scheduled rule (e.g., every 5 minutes)

---

## Purpose

This Lambda polls the **replicated RTS SQL Server database** using the `getDeltaForRelationship`
stored procedure to capture incremental relationship changes and apply them to **Aurora
PostgreSQL** (the TRS primary store).

This is a **polling / watermark-based CDC** approach, complementary to the existing
[RTS Membership Sync Lambda](./RTS_Sync_Lambda.md) which handles push-based webhook events.
Both Lambdas write to the same Aurora tables; the delta sync Lambda acts as a safety net and
handles relationship types that are not delivered via the webhook path.

---

## Architecture

```
┌─────────────────────┐
│  Amazon EventBridge │
│  Scheduled Rule     │
│  (cron: every 5m)   │
└──────────┬──────────┘
           │  Invoke
           ▼
┌─────────────────────────────────────────────────────────┐
│  Lambda: RTS Delta Sync                                 │
│  (.NET 8 / C#)                                          │
│                                                         │
│  For each (client, relationshipType) pair:              │
│    1. Read watermark from Aurora (rts_sync_watermarks)  │
│    2. Call getDeltaForRelationship on SQL Server         │
│    3. Apply delta rows to Aurora relationship tables     │
│    4. Update watermark                                   │
└──────────┬──────────────────────────┬───────────────────┘
           │  EXEC stored procedure   │  UPSERT / DELETE
           ▼                          ▼
┌─────────────────────┐    ┌──────────────────────────────┐
│  Replicated RTS     │    │  Aurora PostgreSQL            │
│  SQL Server         │    │  (TRS primary store)         │
│  (read-only replica)│    │  • roster_student            │
│                     │    │  • school_student            │
│  dbo.tblRelationship│    │  • district_student          │
│  dbo.tblClient      │    │  • district_school           │
│  dbo.tblRelationship│    │  • teacher_roster            │
│       Type          │    └──────────────────────────────┘
└─────────────────────┘
```

---

## The `getDeltaForRelationship` Stored Procedure

```sql
CREATE OR ALTER PROCEDURE dbo.getDeltaForRelationship
    @dateAsOfUtc     DATETIME,
    @relationshipType VARCHAR(255),
    @client          VARCHAR(50)
AS
BEGIN
    DECLARE @clientKey BIGINT;
    SELECT @clientKey = _Key FROM dbo.tblClient WITH (NOLOCK) WHERE Name = @client;

    DECLARE @relationshipTypeKey BIGINT;
    SELECT @relationshipTypeKey = rt._Key
    FROM dbo.tblRelationshipType AS rt WITH (NOLOCK)
    JOIN dbo.tblSetOfClientRelationshipType AS scrt WITH (NOLOCK)
      ON scrt._fk_RelationshipType = rt._Key AND scrt._fk_Client = @clientKey
    WHERE rt.relationshipType = @relationshipType;

    SELECT
        r._fk_Entity_A,
        r._fk_Entity_B,
        CASE WHEN r.endDate > GETDATE() THEN 1 ELSE 0 END AS isDelete
    FROM dbo.tblRelationship AS r WITH (NOLOCK)
    WHERE r._fk_RelationshipType = @relationshipTypeKey
      AND r.UTCDate > @dateAsOfUtc
END
```

### Output columns

| Column | Type | Meaning |
|--------|------|---------|
| `_fk_Entity_A` | BIGINT | "Left" entity key (e.g., roster, school, district, teacher) |
| `_fk_Entity_B` | BIGINT | "Right" entity key (e.g., student) |
| `isDelete` | BIT | `1` = relationship end date is **in the future** (currently active → UPSERT); `0` = end date has **passed** (relationship ended → deactivate) |

> ⚠️ **Note on `isDelete` semantics:** The column name is counterintuitive.  
> `isDelete = 1` means `endDate > GETDATE()` — the relationship is **active**.  
> `isDelete = 0` means `endDate ≤ GETDATE()` — the relationship has **expired or been withdrawn**.  
> The Lambda treats `isDelete = 1` as an **UPSERT** (active) and `isDelete = 0` as a **soft delete**
> (`active = FALSE`).

---

## Relationship Type → PostgreSQL Table Mapping

The procedure is called once per `(client, relationshipType)` pair. The Lambda maps each RTS
relationship type to the corresponding Aurora table:

| RTS `relationshipType` | `_fk_Entity_A` | `_fk_Entity_B` | Target Aurora Table |
|------------------------|----------------|----------------|---------------------|
| `RosterStudent`        | roster_id      | student_id     | `trs.roster_student` |
| `SchoolStudent`        | school_id      | student_id     | `trs.school_student` |
| `DistrictStudent`      | district_id    | student_id     | `trs.district_student` |
| `DistrictSchool`       | district_id    | school_id      | `trs.district_school` |
| `TeacherRoster`        | teacher_id     | roster_id      | `trs.teacher_roster` |

The set of relationship types and client names are provided as Lambda environment variables (or
AWS SSM Parameter Store entries) so no code changes are needed when new clients or relationship
types are added.

---

## Watermark Management

The Lambda stores the last-successfully-processed timestamp per `(tenant_id, relationshipType)` in
a dedicated Aurora table:

```sql
CREATE TABLE trs.rts_sync_watermarks (
    tenant_id         TEXT        NOT NULL,
    relationship_type TEXT        NOT NULL,
    last_synced_utc   TIMESTAMPTZ NOT NULL DEFAULT '1970-01-01T00:00:00Z',
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, relationship_type)
);
```

**Watermark update strategy:**

1. **Read** `last_synced_utc` for `(tenant_id, relationshipType)` before calling the procedure.
2. Record `batch_start_time = NOW()` (captured in Lambda, before the SQL Server call).
3. Call `getDeltaForRelationship` with `@dateAsOfUtc = last_synced_utc`.
4. Apply all delta rows to Aurora.
5. **Only after all rows are committed**, update `last_synced_utc = batch_start_time`.

Using `batch_start_time` (not the max `UTCDate` from results) avoids missing rows written to RTS
between the procedure call and the watermark update.

**Overlap buffer:** Subtract a configurable safety buffer (default: 30 seconds) from
`batch_start_time` when writing the watermark to guard against clock skew between SQL Server and
Lambda:

```
new_watermark = batch_start_time − 30s
```

This means up to 30 seconds of rows may be re-processed on the next run. Aurora UPSERTs are
idempotent, so this is safe.

---

## Processing Flow

```
On EventBridge trigger:

FOR EACH (tenant_id, client_name, relationshipType, targetTable) in config:

  1. READ WATERMARK
     SELECT last_synced_utc FROM trs.rts_sync_watermarks
     WHERE tenant_id = ? AND relationship_type = ?
     -- Default '1970-01-01' on first run (full sync)

  2. CAPTURE batch_start_time = UtcNow()

  3. CALL SQL SERVER
     EXEC dbo.getDeltaForRelationship
       @dateAsOfUtc     = last_synced_utc,
       @relationshipType = relationshipType,
       @client          = client_name

  4. PARTITION RESULTS
     active_rows  = rows WHERE isDelete = 1
     expired_rows = rows WHERE isDelete = 0

  5. UPSERT ACTIVE ROWS (batch, 500 rows per statement)
     INSERT INTO trs.<targetTable>
       (tenant_id, <entity_a_col>, <entity_b_col>, active, synced_at)
     VALUES ...
     ON CONFLICT (<pk_columns>) DO UPDATE SET
       active    = TRUE,
       synced_at = NOW()

  6. DEACTIVATE EXPIRED ROWS (batch, 500 rows per statement)
     INSERT INTO trs.<targetTable>
       (tenant_id, <entity_a_col>, <entity_b_col>, active, synced_at)
     VALUES ...
     ON CONFLICT (<pk_columns>) DO UPDATE SET
       active    = FALSE,
       synced_at = NOW()

  7. UPDATE WATERMARK
     INSERT INTO trs.rts_sync_watermarks (tenant_id, relationship_type, last_synced_utc, updated_at)
     VALUES (?, ?, batch_start_time − interval '30 seconds', NOW())
     ON CONFLICT (tenant_id, relationship_type) DO UPDATE SET
       last_synced_utc = EXCLUDED.last_synced_utc,
       updated_at      = NOW()

  8. EMIT METRICS
     CloudWatch: TRS/RTSDeltaSync/RowsUpserted, TRS/RTSDeltaSync/RowsExpired
```

---

## Aurora UPSERT Patterns per Table

### `trs.roster_student`

```sql
INSERT INTO trs.roster_student (tenant_id, roster_id, student_id, active, synced_at)
VALUES ($tenant, $entity_a::TEXT, $entity_b, TRUE, NOW())
ON CONFLICT (tenant_id, roster_id, student_id) DO UPDATE SET
    active    = EXCLUDED.active,
    synced_at = EXCLUDED.synced_at;
```

### `trs.school_student`

```sql
INSERT INTO trs.school_student (tenant_id, school_id, student_id, active, synced_at)
VALUES ($tenant, $entity_a::TEXT, $entity_b, TRUE, NOW())
ON CONFLICT (tenant_id, school_id, student_id) DO UPDATE SET
    active    = EXCLUDED.active,
    synced_at = EXCLUDED.synced_at;
```

### `trs.district_student`

```sql
INSERT INTO trs.district_student (tenant_id, district_id, student_id, active, synced_at)
VALUES ($tenant, $entity_a::TEXT, $entity_b, TRUE, NOW())
ON CONFLICT (tenant_id, district_id, student_id) DO UPDATE SET
    active    = EXCLUDED.active,
    synced_at = EXCLUDED.synced_at;
```

### `trs.district_school`

```sql
INSERT INTO trs.district_school (tenant_id, district_id, school_id)
VALUES ($tenant, $entity_a::TEXT, $entity_b::TEXT)
ON CONFLICT (tenant_id, district_id, school_id) DO NOTHING;
-- district_school has no active/synced_at; expired rows are hard-deleted:
-- DELETE FROM trs.district_school WHERE tenant_id = $tenant
--   AND district_id = $entity_a AND school_id = $entity_b  [for expired rows]
```

### `trs.teacher_roster`

```sql
INSERT INTO trs.teacher_roster (tenant_id, teacher_id, roster_id)
VALUES ($tenant, $entity_a::TEXT, $entity_b::TEXT)
ON CONFLICT (tenant_id, teacher_id, roster_id) DO NOTHING;
-- Expired rows are hard-deleted (no active column on this table):
-- DELETE FROM trs.teacher_roster WHERE tenant_id = $tenant
--   AND teacher_id = $entity_a AND roster_id = $entity_b  [for expired rows]
```

---

## EventBridge Configuration

```json
{
  "Name": "trs-rts-delta-sync",
  "ScheduleExpression": "rate(5 minutes)",
  "State": "ENABLED",
  "Targets": [
    {
      "Arn": "arn:aws:lambda:<region>:<account>:function:trs-rts-delta-sync",
      "Id": "RTSDeltaSyncTarget"
    }
  ]
}
```

**Lambda timeout:** 5 minutes (matches schedule frequency; single-client runs typically complete in
< 30 seconds; multi-client with large deltas may take 2–3 minutes).

**Concurrency:** Lambda reserved concurrency = 1 per tenant to prevent overlapping executions
writing conflicting watermarks. Use SQS FIFO or EventBridge input transformer to fan-out per
tenant if multiple clients must run in parallel.

---

## Configuration (Environment Variables / SSM)

| Variable | Example | Description |
|----------|---------|-------------|
| `RTS_CONN_STRING_SSM` | `/trs/rts/connstring` | SSM path to SQL Server connection string |
| `AURORA_CONN_STRING_SSM` | `/trs/aurora/connstring` | SSM path to Aurora connection string |
| `SYNC_OVERLAP_SECONDS` | `30` | Watermark safety buffer in seconds |
| `BATCH_SIZE` | `500` | Rows per Aurora batch UPSERT |
| `CLIENTS_CONFIG_SSM` | `/trs/rts/clients` | SSM path to JSON array of `{tenantId, clientName, relationshipTypes[]}` |

**Example `CLIENTS_CONFIG_SSM` value:**
```json
[
  {
    "tenantId": "tx",
    "clientName": "Texas",
    "relationshipTypes": ["RosterStudent", "SchoolStudent", "DistrictStudent", "DistrictSchool", "TeacherRoster"]
  },
  {
    "tenantId": "va",
    "clientName": "Virginia",
    "relationshipTypes": ["RosterStudent", "SchoolStudent", "DistrictStudent"]
  }
]
```

---

## Relationship with Existing RTS Webhook Lambda

| Concern | [RTS_Sync_Lambda](./RTS_Sync_Lambda.md) (webhook) | This Lambda (delta poll) |
|---------|--------------------------------------------------|--------------------------|
| Trigger | Webhook / SQS event from RTS | EventBridge schedule (every 5m) |
| Latency | Near-real-time (seconds) | Up to 5 minutes |
| Coverage | Only events RTS pushes | All changes since watermark |
| Reliability | Depends on RTS push availability | Independent; polls replica |
| Use case | Primary, low-latency path | Safety net; catches missed events; handles back-to-school bulk sync |
| ClickHouse mirror | Writes `student_scope_today` + `student_attributes` | Aurora only (PeerDB handles ClickHouse mirror via WAL) |

Both Lambdas write to the same Aurora tables with idempotent UPSERTs — no coordination needed.
PeerDB's WAL-based CDC picks up all Aurora changes (from either Lambda) and keeps the ClickHouse
membership mirrors current.

---

## Error Handling

| Condition | Action |
|-----------|--------|
| SQL Server connection failure | Retry 3× with exponential backoff; abort run; watermark NOT updated; next scheduled run retries from same watermark |
| Empty result set | Update watermark normally (no rows changed is valid) |
| Aurora write failure (transient) | Retry batch up to 5×; on persistent failure abort and do NOT update watermark |
| Unknown `relationshipType` in config | Log warning and skip; emit `TRS/RTSDeltaSync/UnknownRelationshipType` CloudWatch metric |
| Large delta (> 100k rows) | Process in batches of 500; Lambda timeout risk → consider per-client Step Function for back-to-school full sync |
| Lambda timeout | Watermark was NOT updated for the timed-out batch; next run re-processes from same watermark; safe due to UPSERT idempotency |

---

## IAM Permissions

- Read SSM Parameter Store paths under `/trs/rts/` and `/trs/aurora/`
- Invoke Lambda (EventBridge needs `lambda:InvokeFunction`)
- Read/write Aurora PostgreSQL via Secrets Manager credential
- Write CloudWatch metrics (`cloudwatch:PutMetricData`)

---

## Monitoring

| CloudWatch Metric | Description |
|-------------------|-------------|
| `TRS/RTSDeltaSync/RowsUpserted` | Active rows written per invocation (dimension: `tenantId`, `relationshipType`) |
| `TRS/RTSDeltaSync/RowsExpired` | Deactivated rows per invocation |
| `TRS/RTSDeltaSync/DurationMs` | Wall-clock time per (tenant, relationshipType) batch |
| `TRS/RTSDeltaSync/WatermarkLagSeconds` | Current watermark age — alerts if > 15 minutes indicate stalled sync |
| `TRS/RTSDeltaSync/SQLServerError` | Count of SQL Server call failures |

**Alerting:** PagerDuty/SNS alert when `WatermarkLagSeconds > 900` (15 minutes) or
`SQLServerError > 0` in a 10-minute window.

---

## Back-to-School Full Sync

On first run (watermark = `1970-01-01`) or forced reset, the procedure returns all rows for the
client. For large clients (Texas, ~5.5M students):

1. Reset watermark to `1970-01-01` via one-time script.
2. Increase Lambda timeout to 15 minutes for the initial run (or orchestrate via Step Functions).
3. Process in 500-row batches; Aurora bulk UPSERT: ~30–60 minutes for 5.5M rows.
4. After completion, normal 5-minute schedule resumes.

**Alternative:** Use the existing `RTS_Sync_Lambda` bulk sync path for back-to-school (it is
designed for large batch operations) and rely on this Lambda only for ongoing delta sync.
