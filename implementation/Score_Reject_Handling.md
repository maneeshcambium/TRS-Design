# Score File Rejection Handling — Implementation Spec

**Component:** Lambda: Score Processor  
**Relates to:** TRS Design Document v4 §8.5  
**Status:** Draft

---

## 1. Overview

When the Score Processor Lambda cannot process an S3 score file — because test configuration for
the file's `test_key` does not exist in TRS — it must:

1. Copy the original file to the rejection S3 bucket (preserve it, do **not** delete from source)
2. Record the rejection in the Postgres `score_ingest_rejections` table (audit + resolution tracking)
3. Emit a CloudWatch metric — CloudWatch **alarms** drive all SNS notifications (see §8)
4. **Acknowledge the SQS message** — do not allow it to retry, it will always fail until the config exists

The rejection is a **permanent stop** for that file, not a transient error. Retrying will not help;
only adding the missing `test_key` config and replaying the file will resolve it.

> **Why no per-file SNS:** A single missing `test_key` can produce hundreds of rejected files in
> minutes. Publishing one SNS message per file creates an alert storm with no additional signal.
> CloudWatch metric alarms aggregate across the evaluation window (e.g. count > 0 in 5 min) and
> fire once — that single alarm notification carries the same information without the noise.

---

## 2. Rejection Reasons

| Code | Trigger | Retry? |
|------|---------|--------|
| `UNKNOWN_TEST_CONFIG` | `test_key` in the score file has no matching row in `trs.test_configs` | No — move to rejection bucket; wait for config |
| `VALIDATION_FAILED` | Required fields missing or unparseable JSON | No — malformed file; human review needed |
| `DATA_CONFLICT` | Same `opp_key` + `date_scored` but different score data (hash mismatch) | No — conflicting source data; ops investigation |

---

## 3. S3 Rejection Bucket Layout

Rejection bucket name: `trs-rejected`

```
trs-rejected/
  {tenant_id}/
    {rejection_reason}/     ← e.g., unknown-test-config / validation-failed / data-conflict
      {test_key}/           ← the test_key from the file (or 'unknown' for VALIDATION_FAILED)
        {yyyy-MM-dd}/       ← UTC date of rejection
          {original-filename}
```

**Example:**
```
trs-rejected/tx/unknown-test-config/checkpoint1-grade5-ela-2026/2026-03-04/scores_batch_00412.json
```

Files are retained indefinitely (lifecycle policy: no expiry; gzip compression via S3 bucket policy).

---

## 4. Postgres Schema

```sql
CREATE TABLE trs.score_ingest_rejections (
    rejection_id      UUID         NOT NULL DEFAULT gen_random_uuid(),
    rejected_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    -- Rejection classification
    rejection_reason  TEXT         NOT NULL,  -- 'UNKNOWN_TEST_CONFIG' | 'VALIDATION_FAILED' | 'DATA_CONFLICT'

    -- Source file identity
    source_s3_key     TEXT         NOT NULL,  -- original path in trs-raw bucket
    source_etag       TEXT,                   -- S3 ETag for dedup
    rejected_s3_key   TEXT         NOT NULL,  -- path in trs-rejected bucket

    -- What we know from the file
    tenant_id         TEXT,                   -- null if VALIDATION_FAILED before tenant extracted
    test_key          TEXT,                   -- null if VALIDATION_FAILED before test_key extracted
    test_event        TEXT,
    school_year       SMALLINT,
    record_count      INTEGER,                -- number of score records in the file

    -- Resolution tracking
    resolved          BOOLEAN      NOT NULL DEFAULT FALSE,
    resolved_at       TIMESTAMPTZ,
    resolved_by       TEXT,                   -- 'ops-replay' | 'auto-replay' | username
    resolution_notes  TEXT,

    PRIMARY KEY (rejection_id)
);

-- Fast lookup for ops dashboard: all open rejections per tenant
CREATE INDEX idx_rejections_unresolved
    ON trs.score_ingest_rejections (tenant_id, rejected_at DESC)
    WHERE resolved = FALSE;

-- Lookup by test_key (for bulk replay after config is added)
CREATE INDEX idx_rejections_test_key
    ON trs.score_ingest_rejections (tenant_id, test_key, rejected_at DESC)
    WHERE resolved = FALSE;
```

---

## 5. Score Processor Lambda — Rejection Flow

### 5.1 Where in the Processing Pipeline

The rejection is evaluated at **step 4** of the Score Processor ingest pipeline —
immediately after attempting to resolve `TestKey → test_group_id` from `test_configs`.

```
Step 1: Download S3 file
Step 2: Validate required fields          ← VALIDATION_FAILED if bad JSON / missing fields
Step 3: Idempotency check (score_ingest_log)
Step 4: Resolve test_group_id            ← UNKNOWN_TEST_CONFIG if no match
Step 5: Compute is_aggregate_eligible
Step 6: Conflict guard                   ← DATA_CONFLICT if hash mismatch on same opp_key + date
Step 7: UPSERT → Aurora student_opportunities
...
```

### 5.2 C# Rejection Handler

```csharp
// ScoreRejectionHandler.cs — injected into Score Processor Lambda
public class ScoreRejectionHandler
{
    private readonly IAmazonS3 _s3;
    private readonly IPostgresRepository _pg;
    private readonly IAmazonCloudWatch _cw;
    private readonly ILogger<ScoreRejectionHandler> _logger;
    private readonly TrsConfig _config;

    public async Task HandleRejectionAsync(
        string sourceS3Key,
        string sourceEtag,
        ScoreFileHeader header,   // partially parsed — tenant_id, test_key, school_year, record_count
        RejectionReason reason,
        string details,
        CancellationToken ct = default)
    {
        var date = DateTime.UtcNow.ToString("yyyy-MM-dd");
        var reasonSlug = reason switch
        {
            RejectionReason.UnknownTestConfig => "unknown-test-config",
            RejectionReason.ValidationFailed  => "validation-failed",
            RejectionReason.DataConflict      => "data-conflict",
            _                                 => "other"
        };

        var testKeySlug = string.IsNullOrWhiteSpace(header?.TestKey) ? "unknown" : header.TestKey;
        var tenantId    = header?.TenantId ?? "unknown";
        var fileName    = Path.GetFileName(sourceS3Key);
        var rejectedKey = $"{tenantId}/{reasonSlug}/{testKeySlug}/{date}/{fileName}";

        // 1. Copy file to rejection bucket (preserve original — no delete from source)
        await _s3.CopyObjectAsync(new CopyObjectRequest
        {
            SourceBucket      = _config.RawBucket,       // trs-raw
            SourceKey         = sourceS3Key,
            DestinationBucket = _config.RejectedBucket,  // trs-rejected
            DestinationKey    = rejectedKey
        }, ct);

        // 2. Record in Postgres rejection log
        await _pg.InsertRejectionAsync(new ScoreIngestRejection
        {
            RejectionReason = reason.ToString().ToUpperInvariant(),   // e.g., "UNKNOWN_TEST_CONFIG"
            SourceS3Key     = sourceS3Key,
            SourceEtag      = sourceEtag,
            RejectedS3Key   = rejectedKey,
            TenantId        = header?.TenantId,
            TestKey         = header?.TestKey,
            TestEvent       = header?.TestEvent,
            SchoolYear      = header?.SchoolYear,
            RecordCount     = header?.RecordCount
        }, ct);

        // 3. Emit CloudWatch metric — CloudWatch alarms trigger SNS (not this code)
        //    A single missing test_key can reject hundreds of files; per-file SNS is noisy.
        //    Alarm fires once when metric count crosses threshold in the evaluation window.
        await _cw.PutMetricDataAsync(new PutMetricDataRequest
        {
            Namespace  = "TRS/Ingest",
            MetricData =
            [
                new MetricDatum
                {
                    MetricName = "ScoreFileRejected",
                    Value      = 1,
                    Unit       = StandardUnit.Count,
                    Dimensions =
                    [
                        new Dimension { Name = "TenantId",        Value = tenantId },
                        new Dimension { Name = "RejectionReason", Value = reason.ToString() }
                    ]
                }
            ]
        }, ct);

        // 4. Structured log — queryable via CloudWatch Logs Insights
        _logger.LogWarning(
            "Score file rejected. {Reason} | Tenant={TenantId} | TestKey={TestKey} | " +
            "Source={SourceKey} | Rejected={RejectedKey} | Records={RecordCount}",
            reason, tenantId, header?.TestKey, sourceS3Key, rejectedKey, header?.RecordCount);

        // NOTE: Do NOT throw — caller acknowledges the SQS message normally.
        // Retrying UNKNOWN_TEST_CONFIG will never succeed until the config is added.
        // The file is safely preserved in trs-rejected for replay.
    }
}

public enum RejectionReason
{
    UnknownTestConfig,
    ValidationFailed,
    DataConflict
}
```

### 5.3 Integration in Score Processor

```csharp
// Inside ScoreProcessorLambda.ProcessFileAsync(...)

// Step 2: Parse and validate
ScoreFileHeader header;
try
{
    header = ParseScoreFile(rawJson);
}
catch (ScoreParseException ex)
{
    await _rejectionHandler.HandleRejectionAsync(s3Key, etag, null,
        RejectionReason.ValidationFailed, ex.Message, ct);
    return;  // ACK the SQS message — do not retry
}

// Step 4: Resolve test config
var testConfig = await _configCache.GetTestConfigAsync(header.TenantId, header.TestKey, ct);
if (testConfig is null)
{
    await _rejectionHandler.HandleRejectionAsync(s3Key, etag, header,
        RejectionReason.UnknownTestConfig,
        $"No test_configs row for tenant={header.TenantId}, test_key={header.TestKey}", ct);
    return;  // ACK the SQS message — do not retry
}

// Step 6: Conflict guard
if (hasDataConflict)
{
    await _rejectionHandler.HandleRejectionAsync(s3Key, etag, header,
        RejectionReason.DataConflict,
        $"opp_key={conflictingOppKey} — same key+timestamp, different score data", ct);
    return;
}
```

---

## 6. Test Config Cache

`test_key → test_group_id` lookup should be cached in-process (Lambda static field) to avoid a
Postgres round-trip on every file. Cache miss triggers a DB lookup; if still null, it is a
genuine `UNKNOWN_TEST_CONFIG` rejection.

```csharp
// In-process cache — survives Lambda warm invocations
private static readonly ConcurrentDictionary<string, TestConfig?> _configCache = new();

private async Task<TestConfig?> GetTestConfigAsync(string tenantId, string testKey, CancellationToken ct)
{
    var cacheKey = $"{tenantId}:{testKey}";

    if (_configCache.TryGetValue(cacheKey, out var cached))
        return cached;  // null means "confirmed not found" — still a valid cached result

    var config = await _pg.QueryTestConfigAsync(tenantId, testKey, ct);
    _configCache[cacheKey] = config;  // cache null too — prevents hammering DB for known-missing configs
    return config;
}
```

**Cache TTL:** Lambda container lifetime (~15 min idle). New configs added to Postgres become
visible when the Lambda container is recycled or a forced cold start occurs.

**Config cache invalidation:** Not needed for correctness — if a test config is added to the DB
after a rejection, the replay mechanism (§7) re-queues the file, which will hit a fresh Lambda
container or a stale cache entry. If stale, the file will be rejected again; the cache entry
will naturally expire on the next cold start. For faster propagation, add a Lambda `Function URL`
admin endpoint to flush the cache: `POST /admin/config-cache/flush`.

---

## 7. Replay Path (After Config is Added)

Once the missing test config is added to `trs.test_configs`:

**Step 1 — Find all open rejections for the test:**

```sql
SELECT rejection_id, source_s3_key, rejected_s3_key, rejected_at, record_count
FROM trs.score_ingest_rejections
WHERE tenant_id       = 'tx'
  AND test_key        = 'checkpoint1-grade5-ela-2026'
  AND rejection_reason = 'UNKNOWN_TEST_CONFIG'
  AND resolved        = FALSE
ORDER BY rejected_at;
```

**Step 2 — Re-queue original S3 keys onto the SQS Score Queue:**

```csharp
// Replay Lambda / admin tool — sends original source S3 keys back to the ingest queue
foreach (var rejection in openRejections)
{
    await _sqs.SendMessageAsync(new SendMessageRequest
    {
        QueueUrl    = _config.ScoreQueueUrl,
        MessageBody = JsonSerializer.Serialize(new { S3Key = rejection.SourceS3Key, Etag = rejection.SourceEtag }),
        MessageAttributes = new Dictionary<string, MessageAttributeValue>
        {
            ["ReplayOf"] = new() { DataType = "String", StringValue = rejection.RejectionId.ToString() }
        }
    });
}
```

The existing `score_ingest_log` idempotency check (keyed on `s3_key + etag`) guarantees that
even if the file was partially processed before the rejection, replaying it will not create
duplicate rows.

**Step 3 — Mark rejections resolved** (after confirming SQS messages enqueued):

```sql
UPDATE trs.score_ingest_rejections
SET    resolved          = TRUE,
       resolved_at       = NOW(),
       resolved_by       = 'ops-replay',
       resolution_notes  = 'test_config added; replayed to SQS score queue 2026-03-04'
WHERE  tenant_id         = 'tx'
  AND  test_key          = 'checkpoint1-grade5-ela-2026'
  AND  rejection_reason  = 'UNKNOWN_TEST_CONFIG'
  AND  resolved          = FALSE;
```

---

## 8. CloudWatch Alarms

| Alarm | Metric | Threshold | Action |
|-------|--------|-----------|--------|
| `TRS-UnknownTestConfig-Any` | `TRS/Ingest/ScoreFileRejected` where `RejectionReason=UnknownTestConfig` | Count > 0 in 5 min | SNS → ops email / PagerDuty |
| `TRS-ValidationFailure-High` | `TRS/Ingest/ScoreFileRejected` where `RejectionReason=ValidationFailed` | Count > 3 in 15 min | SNS → urgent ops alert (scoring system may have schema change) |
| `TRS-DataConflict-Any` | `TRS/Ingest/ScoreFileRejected` where `RejectionReason=DataConflict` | Count > 0 in 5 min | SNS → urgent ops alert (upstream data integrity issue) |

---

## 9. Ops Dashboard Query

```sql
-- Open rejections summary — grouped by tenant + reason + test_key
SELECT
    tenant_id,
    rejection_reason,
    test_key,
    COUNT(*)           AS rejected_files,
    MIN(rejected_at)   AS first_seen,
    MAX(rejected_at)   AS last_seen,
    SUM(record_count)  AS total_records_affected
FROM trs.score_ingest_rejections
WHERE resolved = FALSE
GROUP BY tenant_id, rejection_reason, test_key
ORDER BY last_seen DESC;
```

---

## 10. Infrastructure Notes

| Resource | Detail |
|----------|--------|
| Rejection S3 bucket | `trs-rejected` — versioning ON; lifecycle: no expiry; server-side encryption SSE-S3 |
| CloudWatch namespace | `TRS/Ingest` — metric `ScoreFileRejected`, dimensions: `TenantId`, `RejectionReason` |
| SNS topic | `trs-ingest-alerts` — subscriptions: ops email list + PagerDuty; triggered by **CloudWatch alarms only**, not per-file Lambda code |
| IAM: Lambda execution role | Add `s3:CopyObject` on `trs-raw/*` and `s3:PutObject` on `trs-rejected/*`; **no `sns:Publish` needed** |
| SQS DLQ | Existing Score Queue DLQ is **not** used for rejections — rejections are acknowledged normally; DLQ is only for Lambda exceptions and transient failures |
