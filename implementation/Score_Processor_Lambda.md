# Implementation: Score Processor Lambda

**Belongs to:** [TRS ClickHouse Architecture Design](../TRS_ClickHouse_Design.md)  
**Component:** Ingestion Pipeline — AWS Lambda (Node.js 20 or .NET 8)  
**Trigger:** SQS Score Queue (batch size 10, visibility timeout 5 min)

---

## Purpose

The Score Processor Lambda is the first stage of the TRS ingestion pipeline. It receives score file notifications from SQS, downloads raw score JSON from S3, computes derived fields (`is_aggregate_eligible`, `test_group_id`), and writes the canonical record to **Aurora PostgreSQL** — which is the source of truth. PeerDB then mirrors the Postgres write to ClickHouse via CDC.

The Lambda does **not** write directly to ClickHouse. All ClickHouse writes originate from the CDC pipeline.

---

## Processing Flow

```
For each SQS message (one S3 key per message):

  1. DOWNLOAD & VALIDATE
     - Fetch raw score JSON from S3 (s3://trs-raw/scores/{tenantId}/{key})
     - Validate required fields: opp_key, student_id, test_key, opp_status, date_scored
     - If invalid: move to DLQ; acknowledge SQS message; stop.

  2. IDEMPOTENCY CHECK
     - Query Aurora idempotency table:
         SELECT 1 FROM trs.score_ingest_log WHERE s3_key = ? AND etag = ?
       → If record exists (already processed): acknowledge SQS message; stop.
       → If not found: continue.

  3. RESOLVE CONFIGURATION
     - Query Aurora: SELECT test_group_id FROM test_key_config
         WHERE tenant_id = ? AND test_key = ?
       (cached in Lambda memory for 15 min; config rarely changes)

  4. COMPUTE is_aggregate_eligible
     eligible = (opp_status IN ['scored', 'partially_scored'])
                AND (condition_code IS NULL OR condition_code IN ['', 'none'])
     Store as UInt8: 1 = eligible, 0 = not eligible.
     school_year is also resolved here from the score payload (or derived from
     tested_date: academic year = calendar year of the following July 1).
     All subsequent steps carry school_year explicitly.

  5. CONFLICT GUARD (same opp_key, same date_scored, different data)
     - Query Aurora: SELECT opp_status, overall_perf_level, overall_scale_score
                       -- (or a checksum column if added later)
         FROM trs.student_opportunities
         WHERE tenant_id   = ?
           AND school_year = ?     -- partition pruning: hits exactly one partition
           AND opp_key     = ?
           AND date_scored = ?
     - If row exists and data differs from incoming:
         → Log structured warning; emit CloudWatch metric TRS/Scores/DataConflict
         → DO NOT INSERT; acknowledge SQS message; alert on-call.
         → This indicates an upstream scoring system bug (same opp_key + timestamp,
           different score content). Human review required.
     - If row exists and data is identical: 
         → True duplicate delivery; acknowledge SQS and stop (idempotent discard).
     - If no row exists: proceed.

  6. WRITE TO AURORA (source of truth)
     INSERT INTO trs.student_opportunities (
       tenant_id, school_year, test_group_id, student_id, opp_key,
       test_key, test_event, opp_status, condition_code, tested_date,
       date_scored, is_aggregate_eligible,
       overall_scale_score, overall_raw_score, overall_lexile_score,
       overall_quantile_score, overall_perf_level, overall_standard_error,
       raw_s3_key
     )
     ON CONFLICT (tenant_id, school_year, opp_key) DO UPDATE SET
       -- Postgres partitioned tables require the partition key (school_year)
       -- in the unique constraint, so ON CONFLICT must include it.
       date_scored            = EXCLUDED.date_scored,
       is_aggregate_eligible  = EXCLUDED.is_aggregate_eligible,
       overall_scale_score    = EXCLUDED.overall_scale_score,
       overall_perf_level     = EXCLUDED.overall_perf_level,
       -- ... other score fields
     WHERE EXCLUDED.date_scored > trs.student_opportunities.date_scored;

     Also write component scores (standards, RC):
     INSERT INTO student_component_scores (...) ON CONFLICT (...) DO UPDATE ...

  7. WRITE IDEMPOTENCY RECORD
     - Insert into Aurora: INSERT INTO trs.score_ingest_log (s3_key, etag, processed_at)
         VALUES (?, ?, now()) ON CONFLICT (s3_key, etag) DO NOTHING;
     - Rows older than 14 days are cleaned up nightly by pg_cron:
         DELETE FROM trs.score_ingest_log WHERE processed_at < now() - INTERVAL '14 days';

  8. ACKNOWLEDGE SQS MESSAGE
     - Message is removed from queue only after Aurora commit and idempotency write.
```

---

## Rescore Path

Identical to the normal flow above. The Aurora `ON CONFLICT ... WHERE EXCLUDED.date_scored > ...` clause ensures:
- A later rescore (`date_scored = T2 > T1`) replaces the original row in Postgres.
- PeerDB detects the Postgres UPDATE and emits a Before/After CDC event.
- The Sign-Pair Transformer converts the event to a `-1/+1` Bronze insert pair.
- ClickHouse Gold aggregates are corrected atomically.

No special Lambda code path for rescores — Aurora's UPSERT + `date_scored` guard handles it.

---

## Stale Resend Protection

A stale resend is an upstream re-delivery of an older score for the same opportunity (same `opp_key`, `date_scored = T0 < T1` where T1 is the current version):
- Aurora `WHERE EXCLUDED.date_scored > student_opportunities.date_scored` blocks the older row from overwriting the current one.
- The Lambda detects zero rows updated (pg rowcount = 0), logs a metric `TRS/Scores/StaleResendDiscarded`, and acknowledges the SQS message.
- ClickHouse is unaffected — no CDC event is emitted for a no-op Postgres update.

---

## Future: Multi-Opportunity (Retest) Handling

When a student sits the same test a second time (different `opp_key`, same `student_id + test_group_id`):

```
  After Step 6 (Aurora write for the new opp):

  7a. APPLY MULTI-OPP SELECTION RULE
      - Query Aurora for all opps for this student + year:
          SELECT opp_key, date_scored, is_aggregate_eligible
          FROM trs.student_opportunities
          WHERE tenant_id   = ?
            AND school_year = ?     -- partition pruning
            AND student_id  = ?
            AND test_group_id = ?
      - Apply selection rule from test_family_config (e.g. HIGHEST_SCORE or LATEST_DATE)
      - Identify the winning opp; set is_aggregate_eligible = 1 on winner, 0 on all others.
      - UPDATE Aurora for any opp whose eligibility flag changed.
      - Each Aurora UPDATE triggers a CDC event → PeerDB → Transformer → Bronze insert
        with the corrected is_aggregate_eligible value.
```

The `WHERE is_aggregate_eligible = 1` filter on Gold-layer aggregates propagates the update automatically. No ClickHouse-side logic required.

---

## Error Handling

| Condition | Action |
|-----------|--------|
| S3 file not found | Retry up to 3× (object may not be visible yet); after 3 failures → DLQ |
| Aurora insert failure (transient) | Lambda retries via SQS visibility timeout backoff |
| Aurora insert failure (constraint violation) | Log + DLQ; these should not occur if conflict guard works correctly |
| Aurora idempotency write failure | Non-fatal; log warning; message may be reprocessed on re-delivery (idempotent via ON CONFLICT DO NOTHING) |
| SQS DLQ threshold | Alert on-call via CloudWatch alarm |

---

## Performance Characteristics

| Metric | Value |
|--------|-------|
| Concurrency | Up to 200 Lambda concurrency (SQS auto-scaling) |
| Aurora write latency | ~2–5ms per UPSERT (Serverless v2, same-region) |
| Aurora idempotency log write | ~2ms (INSERT ON CONFLICT DO NOTHING) |
| Total Lambda duration per message | ~50–200ms (dominated by S3 download + Aurora write) |
| End-to-end score visibility (roster query) | ~2 minutes (SQS batch window + Lambda + CDC lag) |

---

## Aurora Supporting Tables

```sql
-- student_opportunities: two-level composite partitioning.
--
-- APPLICATION CODE ALWAYS QUERIES THE PARENT TABLE ONLY.
-- Postgres routes transparently to the correct leaf partition at plan time.
-- No dynamic SQL, no tenant-aware table name construction — ever.
--
--   ✅  SELECT ... FROM trs.student_opportunities WHERE tenant_id = ? AND school_year = ?
--   ❌  SELECT ... FROM trs.student_opportunities_tx_2026   <-- never do this
--
-- Postgres evaluates the WHERE clause at parse/plan time, prunes all non-matching
-- partitions, and scans only the single matching leaf. Partition names are a
-- physical storage detail, invisible to the application layer.
--
-- Level 1 — LIST on tenant_id (one sub-table per state client)
--   Rationale: tenant isolation at the storage layer. When a client off-boards,
--   DETACH + DROP removes all their data atomically with no slow DELETE scan.
--   Also keeps the largest tenant (TX 5.5M students) isolated from others.
--
-- Level 2 — LIST on school_year (one leaf partition per tenant × year)
--   Rationale: school_year is present on every query; partition pruning eliminates
--   all prior-year data from the scan automatically.
--
-- Query pattern: WHERE tenant_id = ? AND school_year = ?
--   → planner prunes to exactly ONE leaf partition; PK index lookup within it.
--
-- Postgres requirement: all partition key columns must be part of any unique
--   constraint. PRIMARY KEY (tenant_id, school_year, opp_key) satisfies both levels.

CREATE TABLE trs.student_opportunities (
    tenant_id              TEXT        NOT NULL,   -- L1 partition key
    school_year            SMALLINT    NOT NULL,   -- L2 partition key
    opp_key                UUID        NOT NULL,
    student_id             INTEGER     NOT NULL,
    test_group_id          TEXT        NOT NULL,
    test_key               TEXT        NOT NULL,
    opp_status             TEXT        NOT NULL,
    condition_code         TEXT,
    tested_date            DATE        NOT NULL,
    date_scored            TIMESTAMPTZ NOT NULL,
    is_aggregate_eligible  BOOLEAN     NOT NULL DEFAULT false,
    overall_scale_score    REAL,
    overall_raw_score      INTEGER,
    overall_perf_level     SMALLINT,
    overall_standard_error REAL,
    raw_s3_key             TEXT        NOT NULL,
    ingested_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, school_year, opp_key)
) PARTITION BY LIST (tenant_id);

-- Level 1: one mid-level partition per tenant, sub-partitioned by school_year.
-- Run once when onboarding a new state client.
CREATE TABLE trs.student_opportunities_tx
    PARTITION OF trs.student_opportunities
    FOR VALUES IN ('tx')
    PARTITION BY LIST (school_year);

CREATE TABLE trs.student_opportunities_va
    PARTITION OF trs.student_opportunities
    FOR VALUES IN ('va')
    PARTITION BY LIST (school_year);
-- ... one per tenant at onboarding time ...

-- Level 2: one leaf partition per tenant × year.
-- Run via a pre-season provisioning Lambda before each year's test window opens.
CREATE TABLE trs.student_opportunities_tx_2025
    PARTITION OF trs.student_opportunities_tx FOR VALUES IN (2025);
CREATE TABLE trs.student_opportunities_tx_2026
    PARTITION OF trs.student_opportunities_tx FOR VALUES IN (2026);

CREATE TABLE trs.student_opportunities_va_2025
    PARTITION OF trs.student_opportunities_va FOR VALUES IN (2025);
CREATE TABLE trs.student_opportunities_va_2026
    PARTITION OF trs.student_opportunities_va FOR VALUES IN (2026);

-- Supporting index for student-level multi-opp lookups (retest handling).
-- CREATE INDEX on the PARENT propagates automatically to all existing and future
-- partitions in Postgres 11+ — no per-leaf index DDL needed.
-- tenant_id and school_year are omitted: the planner already knows the partition.
CREATE INDEX trs_student_opp_student_lookup
    ON trs.student_opportunities (student_id, test_group_id);

-- Off-boarding a client (instant — no row-by-row DELETE):
-- ALTER TABLE trs.student_opportunities DETACH PARTITION trs.student_opportunities_tx;
-- DROP TABLE trs.student_opportunities_tx;  -- cascades to _tx_2025, _tx_2026, ...

-- Idempotency log: prevents double-processing of re-delivered SQS messages
CREATE TABLE trs.score_ingest_log (
    s3_key       TEXT        NOT NULL,
    etag         TEXT        NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (s3_key, etag)
);
-- Cleanup: pg_cron job nightly
-- DELETE FROM trs.score_ingest_log WHERE processed_at < now() - INTERVAL '14 days';
```
