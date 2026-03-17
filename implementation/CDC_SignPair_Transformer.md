# Implementation: C# Sign-Pair Transformer (Fargate Service)

**Belongs to:** [TRS ClickHouse Architecture Design](../TRS_ClickHouse_Design.md)  
**Component:** CDC Pipeline — Fargate `BackgroundService`  
**Runtime:** .NET 8 / C# on AWS Fargate (0.5 vCPU / 1 GB)

---

## Purpose

The Sign-Pair Transformer is the mathematical bridge between the CDC stream (PeerDB or Debezium/Kafka) and the ClickHouse Bronze layer. It is responsible for converting raw PostgreSQL WAL change events into sign-paired row insertions that allow ClickHouse `VersionedCollapsingMergeTree` to resolve rescores atomically.

Without this step, an UPDATE event from the WAL would only carry the new (After) values. The old contribution to Gold-layer aggregates would never be subtracted, causing permanent silent corruption in school and district totals.

---

## Input Contract

The transformer consumes PeerDB webhook/queue events (or Kafka messages in the Debezium variant). Each message is a structured CDC event with the following payload shape:

```
CdcEvent {
    EventType:   INSERT | UPDATE | DELETE
    TableName:   string           // e.g. "trs.student_opportunities"
    Timestamp:   DateTime         // PeerDB event delivery time
    BeforeImage: Row | null       // null for INSERT; populated for UPDATE and DELETE
    AfterImage:  Row | null       // null for DELETE; populated for INSERT and UPDATE
}

Row {
    OppKey:              Guid
    TenantId:            string
    SchoolYear:          ushort
    TestGroupId:         string
    StudentId:           int
    ScoreValue:          float?
    OverallPerfLevel:    byte?
    DateScored:          DateTime64   // millisecond precision — REQUIRED
    IsAggregateEligible: byte         // 1 = eligible, 0 = not
    IsDeleted:           byte         // 0 for normal rows
    // ... all other score columns
}
```

---

## Sign-Pair Emission Logic

### INSERT Event

```
Input:  BeforeImage = null, AfterImage = A

Output: One Bronze row
  → (opp_key=A.OppKey, sign=+1, is_deleted=0, date_scored=A.DateScored, ...A fields)
```

No undo row — there is no prior state to subtract.

---

### UPDATE Event (Rescore)

```
Input:  BeforeImage = B (old score), AfterImage = A (new score)

Validate: A.DateScored > B.DateScored
  If NOT: log warning "out-of-order update detected"; emit pair anyway (VersionedCollapsing handles order)

Output: Two Bronze rows (bundled in a SINGLE batch INSERT to ClickHouse)
  → Row 1 (Undo):  (opp_key=B.OppKey, sign=-1, is_deleted=0, date_scored=B.DateScored, ...B fields)
  → Row 2 (Redo):  (opp_key=A.OppKey, sign=+1, is_deleted=0, date_scored=A.DateScored, ...A fields)
```

**Critical:** Both rows MUST be submitted in the same HTTP batch INSERT to ClickHouse. Splitting them across two requests creates a window where aggregates are temporarily incorrect. ClickHouse processes a single INSERT atomically.

---

### DELETE Event

```
Input:  BeforeImage = B (the row being deleted), AfterImage = null

Output: One Bronze row (Undo only — no Redo counterpart)
  → (opp_key=B.OppKey, sign=-1, is_deleted=1, date_scored=T_now, ...B fields)
```

`T_now` is `DateTime.UtcNow` with millisecond precision — effectively a synthetic "last known event time" ensuring the delete wins against any prior insert in `argMaxState(is_deleted, date_scored)` logic in the Silver layer.

**Note:** `is_deleted = 1` on a `sign = -1` row serves both the Bronze collapse (physical removal) and the Silver layer sentinel (soft-delete visibility filter).

---

### DELETE → Re-Insert Sequence

This sequence is idempotent by design. No special handling is required:

```
T1: INSERT arrives   → emit sign=+1, is_deleted=0, date_scored=T1
T2: DELETE arrives   → emit sign=-1, is_deleted=1, date_scored=T2  (T2 > T1)
T3: Re-INSERT arrives→ emit sign=+1, is_deleted=0, date_scored=T3  (T3 > T2)
```

Silver layer `argMaxState(is_deleted, date_scored)` automatically returns `0` at T3 — the row becomes visible again. Gold-layer aggregates gain the score back via the `+1` sign.

---

## Idempotency Requirements

PeerDB guarantees at-least-once delivery. The transformer may receive the same CDC event twice (e.g. after a Fargate task restart before the last LSN was committed).

Duplicate handling:
- A duplicate INSERT event emits a second `sign=+1` for the same `(opp_key, date_scored)`.
- `VersionedCollapsingMergeTree` treats two rows with the same version key and same sign as "unpaired" — they do NOT cancel. A query with `FINAL` or `WHERE sign = 1` returns them both.
- **Mitigation:** The transformer tracks a small in-memory LRU deduplication window (last N LSN event IDs). For events outside the window, rely on the Aurora-backed idempotency check in the Score Processor Lambda (primary dedup point — see `trs.score_ingest_log`).

For UPDATE events: a duplicate delivery emits a second `-1/+1` pair. Net effect at Bronze: a `-1` and an extra `+1` remain unpaired → Silver `argMax` still returns the correct latest value, but Bronze accumulates phantom rows that background merge cannot collapse. Mitigation: same LSN dedup window.

---

## Bronze Insert Payload

Each ClickHouse Bronze INSERT uses the native binary protocol or the HTTP interface with `FORMAT RowBinary`. A batch of rows for one transaction is submitted as a single HTTP POST to:

```
POST http://{clickhouse-host}:8123/?query=INSERT+INTO+trs.student_scores_bronze+FORMAT+RowBinary
```

Connection pooling: maintain a persistent `HttpClient` pool with a 30-second keepalive. ClickHouse's HTTP interface is stateless — no login round-trip per batch.

---

## Error Handling & Retry

| Error | Action |
|-------|--------|
| ClickHouse unreachable (5xx, connection refused) | Back off exponentially (1s → 2s → 4s → max 30s); do NOT advance PeerDB LSN checkpoint until INSERT succeeds |
| ClickHouse 4xx (schema mismatch) | Log as fatal; emit CloudWatch alarm `TRS/CDC/SchemaError`; do NOT retry (human review required) |
| BeforeImage.DateScored missing or null | Log structured warning; use `1970-01-01T00:00:00.000` as sentinel; emit CloudWatch metric `TRS/CDC/NullDateScored` |
| Malformed event (JSON deserialization failure) | Log + dead-letter to SQS DLQ; advance LSN past the bad event |

---

## Prerequisites

1. `REPLICA IDENTITY FULL` must be set on all replicated Aurora Postgres tables — otherwise DELETE and UPDATE Before images contain only primary key columns (all other fields are null), making the undo row useless for aggregate subtraction.

```sql
ALTER TABLE trs.student_opportunities REPLICA IDENTITY FULL;
ALTER TABLE trs.student_scores        REPLICA IDENTITY FULL;
```

2. `date_scored` column must be `DateTime64(3)` (millisecond precision) in both Postgres and the Bronze schema. `DateTime` (second precision) creates 1-second tie windows in `argMaxState` resolution.

---

## Deployment

| Parameter | Value |
|-----------|-------|
| Container image | `trs/cdc-transformer:latest` (.NET 8 `BackgroundService`) |
| Fargate task size | 0.5 vCPU / 1 GB RAM |
| Scaling | Single task (stateless w.r.t. ClickHouse; PeerDB handles ordering) |
| Environment variables | `PEERDB_SQS_QUEUE_URL`, `CLICKHOUSE_ENDPOINT`, `CLICKHOUSE_USER`, `CLICKHOUSE_PASSWORD` |
| IAM | Read SQS + delete SQS + Secrets Manager (ClickHouse creds) |
