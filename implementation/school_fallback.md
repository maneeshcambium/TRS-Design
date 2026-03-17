
### 9.3 C# Fallback Implementation (School scope)

```csharp
public async Task<AggregateResult> GetSchoolAggregateAsync(
    string tenantId, string schoolId, string testGroupId, int schoolYear,
    IReadOnlyList<string> callerRoles)
{
    // Embargo pre-check (before any cache or DB query)
    await _embargoService.AssertVisibleAsync(tenantId, testGroupId, callerRoles);
    // Throws EmbargoException (→ 404) if embargoed and caller lacks EMBARGO_VIEWER role.
    // embargo_until is cached in-process for 60 s to avoid a DB round-trip per request.

    // Tier 1: Postgres report_cache
    var cached = await _pgRepo.GetReportCacheAsync(
        $"school#{tenantId}#{schoolId}#{schoolYear}#{testGroupId}");
    if (cached != null && cached.ExpiresAt > DateTime.UtcNow)
        return cached.Payload;

    // Tier 2: ClickHouse Gold layer
    if (await _clickHouseHealth.IsAvailableAsync())
    {
        try
        {
            var chResult = await _chRepo.GetSchoolGoldAggregateAsync(
                tenantId, schoolId, testGroupId, schoolYear);
            // Async background cache write — do not block the response
            _ = _pgRepo.UpsertReportCacheAsync(
                $"school#{tenantId}#{schoolId}#{schoolYear}#{testGroupId}",
                chResult, computedBy: "clickhouse_gold", ttlMinutes: 15);
            return chResult;
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "ClickHouse unavailable; falling back to Postgres");
            _metrics.Increment("TRS/API/ClickHouseFallback");
        }
    }

    // Tier 3a: Postgres MV
    var mvResult = await _pgRepo.GetSchoolFromMvAsync(tenantId, schoolId, testGroupId, schoolYear);
    if (mvResult != null)
    {
        _ = _pgRepo.UpsertReportCacheAsync(
            $"school#{tenantId}#{schoolId}#{schoolYear}#{testGroupId}",
            mvResult, computedBy: "postgres_mv", ttlMinutes: 15);
        return mvResult;
    }

    // Tier 3b: Postgres live query
    var pgResult = await _pgRepo.GetSchoolAggregateLiveAsync(
        tenantId, schoolId, testGroupId, schoolYear);
    _ = _pgRepo.UpsertReportCacheAsync(
        $"school#{tenantId}#{schoolId}#{schoolYear}#{testGroupId}",
        pgResult, computedBy: "postgres_live", ttlMinutes: 15);
    return pgResult;
}
```
