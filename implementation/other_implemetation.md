
### 18.1 Key NuGet Packages

```xml
<PackageReference Include="Npgsql"                       Version="8.*" />
<PackageReference Include="ClickHouse.Client"            Version="7.*" />
<PackageReference Include="Amazon.Lambda.Core"           Version="2.*" />
<PackageReference Include="Amazon.Lambda.SQSEvents"      Version="3.*" />
<PackageReference Include="AWSSDK.S3"                    Version="3.*" />
<PackageReference Include="AWSSDK.SQS"                   Version="3.*" />
<PackageReference Include="AWSSDK.SecretsManager"        Version="3.*" />
<PackageReference Include="Amazon.CDK.Lib"               Version="2.*" />
<PackageReference Include="Microsoft.Extensions.Hosting" Version="8.*" />
```

### 18.2 Postgres Batch UPSERT Pattern (C#)

```csharp
// Set-based batch UPSERT using UNNEST — avoids N individual round-trips
var upsertSql = @"
INSERT INTO trs.student_opportunities
    (tenant_id, school_year, opp_key, test_group_id, student_id,
     date_scored, is_aggregate_eligible, overall_scale_score, overall_perf_level, raw_s3_key)
SELECT * FROM UNNEST(
    $1::text[], $2::smallint[], $3::uuid[], $4::text[], $5::int[],
    $6::timestamptz[], $7::boolean[], $8::real[], $9::smallint[], $10::text[])
ON CONFLICT (tenant_id, school_year, opp_key) DO UPDATE SET
    date_scored           = EXCLUDED.date_scored,
    is_aggregate_eligible = EXCLUDED.is_aggregate_eligible,
    overall_scale_score   = EXCLUDED.overall_scale_score,
    overall_perf_level    = EXCLUDED.overall_perf_level
WHERE EXCLUDED.date_scored > trs.student_opportunities.date_scored";
```

### 18.3 ClickHouse Bronze Batch INSERT (C# — Sign-Pair Transformer)

```csharp
// Single HTTP batch INSERT for a -1/+1 pair (or single row for INSERT/DELETE events)
using var client = new ClickHouseClient(clickhouseEndpoint);

var rows = new List<BronzeRow>();

// For an UPDATE: emit both undo (-1) and redo (+1) in the SAME batch
rows.Add(new BronzeRow { OppKey = before.OppKey, Sign = -1, IsDeleted = 0, DateScored = before.DateScored, /* ...before fields */ });
rows.Add(new BronzeRow { OppKey = after.OppKey,  Sign = +1, IsDeleted = 0, DateScored = after.DateScored,  /* ...after fields  */ });

var bulkCopy = new ClickHouseBulkCopy(client)
{
    DestinationTableName = "trs.student_scores_bronze",
    BatchSize = rows.Count  // flush immediately to ensure atomicity of -1/+1 pair
};
await bulkCopy.InitAsync();
await bulkCopy.WriteToServerAsync(rows);
// Do NOT advance PeerDB LSN checkpoint until this INSERT succeeds
```
