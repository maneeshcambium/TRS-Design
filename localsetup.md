# TRS — Local Developer Environment Setup

**Audience:** All TRS developers  
**Purpose:** Run the full TRS stack on a local workstation for development and testing. S3 and SQS use each developer's own personal dev-account resources — no LocalStack required.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [Container Stack](#3-container-stack)
4. [Docker Compose Configuration](#4-docker-compose-configuration)
5. [Schema Initialization](#5-schema-initialization)
6. [Environment Variables](#6-environment-variables)
7. [DevDataSeeder CLI](#7-devdataseeder-cli)
8. [How CDC Is Handled Locally](#8-how-cdc-is-handled-locally)
9. [Mock External Services](#9-mock-external-services)
10. [RTS Roster Data (Direct Seeding)](#10-rts-roster-data-direct-seeding)
11. [Starting Local Dev](#11-starting-local-dev)
12. [Resetting Everything](#12-resetting-everything)
13. [WBS Task Reference](#13-wbs-task-reference)

---

## 1. Overview

The local dev stack replaces all AWS infrastructure so developers can work fully offline. The architecture changes in one key way compared to production:

**Production:** Postgres → PeerDB → SQS → Sign-Pair Transformer → ClickHouse Bronze → Silver/Gold MVs  
**Local dev:** Skip PeerDB entirely. `DevDataSeeder` writes directly to both Postgres and ClickHouse Bronze. ClickHouse materialized views then auto-populate Silver and Gold. RTS roster data is also seeded directly into Postgres — no MSSQL required.

S3 and SQS are **not** emulated locally. Each developer uses their own S3 bucket and SQS queue in the shared TRS dev AWS account (or their personal AWS account). This keeps the Score Processor working against real AWS primitives without the overhead of maintaining LocalStack.

---

## 2. Prerequisites

| Tool | Version | Notes |
|---|---|---|
| Docker Desktop | 4.x+ | Ensure at least 8 GB RAM allocated to Docker |
| Docker Compose | v2.x | Bundled with Docker Desktop |
| .NET SDK | 8.0 | For running API projects and DevDataSeeder |
| Node.js | 20.x | For the React frontends |
| `dotnet-ef` global tool | Latest | `dotnet tool install -g dotnet-ef` |
| AWS CLI | 2.x | For interacting with your personal dev S3 bucket and SQS queue |

---

## 3. Container Stack

> **Key prefix contract:** The `TRS.Caching` library reads `Valkey__KeyPrefix` at startup and prepends it to every key it writes or reads. Format: `{prefix}:{original-key}`. In production and test environments this is empty (or set to `test:` for the test env). On a local workstation it is set to the developer's machine name, e.g. `JOHNS-LAPTOP:school:in:school-101:2026:ela`.



| Container | Image | Port(s) | Purpose |
|---|---|---|---|
| `postgres` | `postgres:16` | 5432 | Primary database — source of truth |
| `clickhouse` | `clickhouse/clickhouse-server:24.3` | 8123 (HTTP), 9000 (native) | Analytical query layer |
| `wiremock` | `wiremock/wiremock:3.5` | 9090 | Stubs for CSR, FlightPlan, ITS, ReportsHub, CRS |

> **Memory budget:** All containers together use approximately 1–2 GB RAM. Ensure Docker Desktop has at least 3 GB allocated.

> **Valkey:** Not containerized. All developers connect to the shared **test-environment Valkey** instance. Each developer prefixes their keys with their machine name to avoid collisions (see [Section 6](#6-environment-variables)).

> **S3 + SQS:** Not containerized. Each developer creates their own S3 bucket (`trs-scores-raw-<initials>`) and SQS queue (`score-ingest-<initials>`) in the shared TRS dev AWS account, then sets the connection details in `local.env`.

---

## 4. Docker Compose Configuration

Save as `docker-compose.local.yml` at the monorepo root.

```yaml
version: "3.9"

services:

  # ─── Postgres ──────────────────────────────────────────────────────────────
  postgres:
    image: postgres:16
    container_name: trs-postgres
    environment:
      POSTGRES_DB: trs
      POSTGRES_USER: trs_app
      POSTGRES_PASSWORD: localpassword
    command: >
      postgres
        -c wal_level=logical
        -c max_replication_slots=5
        -c max_wal_senders=5
        -c max_slot_wal_keep_size=1024
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./infra/sql/postgres/init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U trs_app -d trs"]
      interval: 5s
      timeout: 5s
      retries: 10

  # ─── ClickHouse ────────────────────────────────────────────────────────────
  clickhouse:
    image: clickhouse/clickhouse-server:24.3
    container_name: trs-clickhouse
    ports:
      - "8123:8123"   # HTTP interface (used by .NET ClickHouse.Client)
      - "9000:9000"   # Native protocol
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - ./infra/sql/clickhouse:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8123/ping || exit 1"]
      interval: 5s
      timeout: 5s
      retries: 10
    ulimits:
      nofile:
        soft: 262144
        hard: 262144

  # ─── WireMock (External service stubs) ────────────────────────────────────
  wiremock:
    image: wiremock/wiremock:3.5
    container_name: trs-wiremock
    ports:
      - "9090:8080"
    volumes:
      - ./dev/wiremock:/home/wiremock:ro
    command: --verbose

volumes:
  postgres_data:
  clickhouse_data:
```

---

## 5. Schema Initialization

### Postgres — Flyway migrations auto-applied on first `docker compose up`

The `postgres` container mounts `./infra/sql/postgres/init/` into `/docker-entrypoint-initdb.d/`. Files are executed in alphabetical order on first container creation. Name migration files with the same `V###__` convention as Flyway:

```
infra/sql/postgres/init/
  V001__create_schema.sql          ← trs schema + student_opportunities partition table
  V002__membership_tables.sql      ← roster_student, school_student, district_student, etc.
  V003__config_tables.sql          ← test_aliases, test_keys, embargo_roles, etc.
  V004__operational_tables.sql     ← score_ingest_log, report_cache, etc.
  V005__materialized_views.sql     ← mv_school_overall, mv_district_overall, etc.
  V006__indexes.sql                ← all secondary indexes
  V999__seed_tenant.sql            ← INSERT one tenant row (tenant_id='in', name='Indiana')
```

> **Note:** `docker-entrypoint-initdb.d` only runs when the Postgres data volume is empty (first start). To re-run migrations, [reset the environment](#12-resetting-everything).

### ClickHouse — DDL applied via init scripts

ClickHouse also supports `/docker-entrypoint-initdb.d/`. Place DDL scripts there in deployment order (matching the order specified in `Schema_ClickHouse.md`):

```
infra/sql/clickhouse/
  01_bronze.sql       ← student_scores_bronze, student_component_scores_bronze
  02_silver.sql       ← student_scores_silver + silver_mv
  03_mirrors.sql      ← membership_school_mirror, membership_district_mirror
  04_gold.sql         ← school_aggregates_gold, district_aggregates_gold
  05_gold_mvs.sql     ← bronze_to_school_gold_mv, bronze_to_district_gold_mv
```

### AWS Dev Resources — One-time per developer setup

Each developer creates their own S3 bucket and SQS queue in the TRS dev AWS account (or their personal account). Run once:

```bash
# Replace <initials> with your initials, e.g. jd
INITIALS=<initials>
REGION=us-east-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create S3 buckets
aws s3 mb s3://trs-scores-raw-${INITIALS} --region ${REGION}
aws s3 mb s3://trs-scores-rejected-${INITIALS} --region ${REGION}

# Create SQS queues
aws sqs create-queue --queue-name score-ingest-${INITIALS} \
  --attributes VisibilityTimeout=300 --region ${REGION}

aws sqs create-queue --queue-name score-ingest-dlq-${INITIALS} --region ${REGION}

# Wire S3 → SQS event notification
QUEUE_ARN=arn:aws:sqs:${REGION}:${ACCOUNT_ID}:score-ingest-${INITIALS}

aws s3api put-bucket-notification-configuration \
  --bucket trs-scores-raw-${INITIALS} \
  --notification-configuration "{
    \"QueueConfigurations\": [{
      \"QueueArn\": \"${QUEUE_ARN}\",
      \"Events\": [\"s3:ObjectCreated:*\"],
      \"Filter\": {\"Key\": {\"FilterRules\": [{\"Name\": \"prefix\", \"Value\": \"scores/\"}]}}
    }]
  }"

echo "Dev AWS resources ready. Update local.env with your bucket/queue names."
```

After running, update the `AWS__*` values in your `local.env` with your `<initials>`-suffixed names.

---

## 6. Environment Variables

Create a `local.env` file at the monorepo root by copying the example file:

```bash
cp local.env.example local.env
```

Never commit `local.env`. It is already in `.gitignore`.

### `local.env.example`

```dotenv
# ─── Environment mode ──────────────────────────────────────────────────────
DEV_MODE=true                        # Disables real SSO JWT validation; uses fixed test token

# ─── Postgres ───────────────────────────────────────────────────────────────
ConnectionStrings__Postgres=Host=localhost;Port=5432;Database=trs;Username=trs_app;Password=localpassword

# ─── ClickHouse ─────────────────────────────────────────────────────────────
ClickHouse__Url=http://localhost:8123
ClickHouse__Database=trs

# ─── Valkey (shared test-env instance) ─────────────────────────────────────
# Ask the architect for the test Valkey endpoint and auth token.
Valkey__ConnectionString=<test-valkey-endpoint>:6379,password=<test-valkey-token>,ssl=true
# Your machine key prefix — prevents collisions with other developers and test env data.
# Use your machine hostname or initials. Example: jd-dev or JOHNS-LAPTOP
Valkey__KeyPrefix=<your-machine-name>

# ─── AWS dev S3 + SQS (your personal dev resources — see Section 5) ────────
# Replace <initials> with your own. Ask architect for the TRS dev account ID.
AWS__Region=us-east-1
AWS__AccessKey=<your-dev-access-key>
AWS__SecretKey=<your-dev-secret-key>
AWS__ScoresBucket=trs-scores-raw-<initials>
AWS__RejectedBucket=trs-scores-rejected-<initials>
AWS__ScoreIngestQueueUrl=https://sqs.us-east-1.amazonaws.com/<account-id>/score-ingest-<initials>

# ─── External service stubs (WireMock) ──────────────────────────────────────
ExternalServices__CsrBaseUrl=http://localhost:9090/csr
ExternalServices__FlightPlanBaseUrl=http://localhost:9090/flightplan
ExternalServices__ItsBaseUrl=http://localhost:9090/its
ExternalServices__ReportsHubBaseUrl=http://localhost:9090/reportshub
ExternalServices__CrsBaseUrl=http://localhost:9090/crs

# ─── Dev auth bypass ────────────────────────────────────────────────────────
# When DEV_MODE=true, the Lambda Authorizer is bypassed.
# Use this fixed bearer token in requests: "Bearer dev-local-token"
# The token is mapped to the following test identity:
DevAuth__TenantId=in
DevAuth__UserId=dev-user-1
DevAuth__Roles=TrsUser,TrsAdmin
```

---

## 7. DevDataSeeder CLI

The `DevDataSeeder` is a .NET CLI tool that populates both Postgres and ClickHouse Bronze with realistic fake data. It lives at `src/tools/DevDataSeeder/`.

### What it seeds

| Step | What | Target |
|---|---|---|
| 1 | Reference data — 1 tenant (`in`), test aliases, test keys, performance level configs | Postgres config tables |
| 2 | Roster hierarchy — 1 district → N schools → M teachers → K students per teacher | Postgres membership tables |
| 3 | Embargo config — 1 sample embargo row (inactive by default) | Postgres `embargo_roles` |
| 4 | Fake opportunities — generates `--count` opp records per student with realistic field values | Postgres `student_opportunities` |
| 5 | ClickHouse Bronze direct-write — same opp records inserted as `sign=+1` Bronze rows | ClickHouse `student_scores_bronze` |

Steps 4 and 5 write the same data in parallel, bypassing PeerDB. ClickHouse MVs then automatically populate Silver and Gold.

### Usage

```bash
# Seed with defaults (500 students, 3 schools, ~5 opps each)
dotnet run --project src/tools/DevDataSeeder -- --env local.env

# Custom scale
dotnet run --project src/tools/DevDataSeeder -- \
  --env local.env \
  --students 2000 \
  --schools 10 \
  --opps-per-student 8 \
  --school-year 2026

# Seed only config/roster (skip opps — useful for Admin UI development)
dotnet run --project src/tools/DevDataSeeder -- \
  --env local.env \
  --skip-opps

# Reset and re-seed (drops all trs.student_opportunities rows first)
dotnet run --project src/tools/DevDataSeeder -- \
  --env local.env \
  --reset
```

### Fake opp field rules

| Field | Generation rule |
|---|---|
| `opp_key` | `Guid.NewGuid()` |
| `tenant_id` | Always `in` |
| `school_year` | Configurable via `--school-year` (default: 2026) |
| `student_id` | Sequential integer starting from 100001 |
| `test_key` | Random pick from seeded test keys in config |
| `test_group_id` | Derived from test key config (same logic as Score Processor) |
| `opp_status` | 85% `scored`, 10% `partially_scored`, 5% `invalidated` |
| `condition_code` | 90% `null`, 5% `none`, 5% `SD` |
| `tested_date` | Random date within the current school year |
| `date_scored` | `tested_date + random hours`, millisecond precision |
| `overall_scale_score` | Normal distribution around 2500 (SD=150) |
| `overall_perf_level` | Derived from score using seeded cut scores |
| `is_aggregate_eligible` | Computed same way as Score Processor (opp_status + condition_code) |

---

## 8. How CDC Is Handled Locally

In production, the data flow is:

```
Postgres write → WAL → PeerDB → SQS → Sign-Pair Transformer → ClickHouse Bronze
```

Locally, **PeerDB is not running**. The `DevDataSeeder` replaces the entire CDC pipeline by writing directly to both stores:

```
DevDataSeeder → Postgres student_opportunities  (source of truth)
DevDataSeeder → ClickHouse student_scores_bronze (sign=+1 rows)
                    ↓ (automatic, via MV)
              ClickHouse student_scores_silver
                    ↓ (automatic, via MV)
              ClickHouse school_aggregates_gold
              ClickHouse district_aggregates_gold
```

### What this means for developers

- **TRS Reporting API developers:** Gold layer is fully populated. All three-tier fallback paths work: Valkey (warm after first request) → ClickHouse Gold → Postgres MV.
- **Score Processor developers:** You can still test the real Score Processor by dropping a file into your dev S3 bucket (`s3://trs-scores-raw-<initials>/scores/in/test.json`). The Score Processor picks it up via SQS and writes to Postgres. However, ClickHouse will **not** be updated (no PeerDB). Re-run DevDataSeeder with `--reset` to refresh ClickHouse after a testing session.
- **CDC / Sign-Pair Transformer developers:** Run each component manually and point them at the local Postgres and ClickHouse containers. No special steps needed.

---

## 9. Mock External Services (WireMock)

WireMock stubs live in `dev/wiremock/mappings/`. Each file is a JSON stub mapping.

```
dev/wiremock/
  mappings/
    csr-get-publications.json
    csr-get-standards.json
    flightplan-get-tests.json
    its-get-pld-measures.json
    its-get-cut-scores.json
    reportshub-generate-isr.json
    crs-generate-sdf.json
  __files/
    sample-isr.pdf           ← Binary fixture returned by ReportsHub stub
    sample-sdf.xlsx          ← Binary fixture returned by CRS stub
```

### Example stub: CSR get publications

```json
{
  "request": {
    "method": "GET",
    "urlPathPattern": "/csr/publications"
  },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json" },
    "jsonBody": {
      "publications": [
        { "id": "pub-ela-2026", "name": "ELA Standards 2026", "subject": "ELA" },
        { "id": "pub-math-2026", "name": "Math Standards 2026", "subject": "Math" }
      ]
    }
  }
}
```

### Testing circuit breaker / failure paths

To simulate a downed external service, use the WireMock admin API:

```bash
# Make ReportsHub return 503
curl -X POST http://localhost:9090/__admin/mappings \
  -H "Content-Type: application/json" \
  -d '{
    "request": { "method": "ANY", "urlPathPattern": "/reportshub/.*" },
    "response": { "status": 503, "body": "Service Unavailable" }
  }'

# Reset to normal stubs
curl -X POST http://localhost:9090/__admin/mappings/reset
```

---

## 10. RTS Roster Data (Direct Seeding)

There is no MSSQL container in the local stack. The `RtsSyncWorker` (which reads from MSSQL) is not run locally. Instead, `DevDataSeeder` pumps roster/membership data **directly into Postgres** — the same approach used for fake opps.

This means all Postgres membership tables are pre-populated by the seeder:

| Postgres table | Seeded by |
|---|---|
| `trs.roster_student` | DevDataSeeder — teacher → student assignments |
| `trs.school_student` | DevDataSeeder — school → student enrollment |
| `trs.district_student` | DevDataSeeder — district → student enrollment |
| `trs.district_school` | DevDataSeeder — district → school hierarchy |
| `trs.teacher_roster` | DevDataSeeder — teacher → roster mapping |

The seeder generates a coherent hierarchy (1 district → N schools → M teachers per school → K students per teacher) so that all TRS Reporting API roster/school/district queries return meaningful data.

```bash
# Seed roster only (skip opps — useful when only testing roster/membership queries)
dotnet run --project src/tools/DevDataSeeder -- --env local.env --skip-opps

# Seed everything including opps (default)
dotnet run --project src/tools/DevDataSeeder -- --env local.env

# Verify membership data was seeded
psql postgresql://trs_app:localpassword@localhost:5432/trs \
  -c "SELECT COUNT(*) FROM trs.school_student WHERE tenant_id = 'in';"
```

**Developing the RtsSyncWorker itself?** Point `ConnectionStrings__RtsMssql` in your `local.env` at any accessible SQL Server instance (e.g., the shared dev RTS replica). The worker runs fine against a real MSSQL source — you just don't need one for general TRS development.

---

## 11. Starting Local Dev

### First time

```bash
# 1. Clone and navigate to monorepo root
git clone <repo-url> trs-platform
cd trs-platform

# 2. Copy environment file
cp local.env.example local.env

# 3. Start all containers (Postgres, ClickHouse, Valkey, WireMock)
docker compose -f docker-compose.local.yml up -d

# 4. Wait for all containers to be healthy (~30–60s)
docker compose -f docker-compose.local.yml ps

# 5. Seed reference data, roster, and fake opportunities
dotnet run --project src/tools/DevDataSeeder -- --env local.env

# 6. Start the API you're working on
cd src/trs-api
dotnet run --launch-profile Local

# OR for Admin API
cd src/admin-api
dotnet run --launch-profile Local
```

### Day-to-day

```bash
# Containers already running? Just start your API.
dotnet run --project src/trs-api --launch-profile Local

# Containers stopped? Restart (data volumes preserved)
docker compose -f docker-compose.local.yml up -d
```

---

## 12. Resetting Everything

```bash
# Stop all containers and delete ALL data volumes (full reset)
docker compose -f docker-compose.local.yml down -v

# Restart fresh and re-seed
docker compose -f docker-compose.local.yml up -d
dotnet run --project src/tools/DevDataSeeder -- --env local.env

# Reset only ClickHouse data (keep Postgres intact)
docker compose -f docker-compose.local.yml stop clickhouse
docker volume rm trs-platform_clickhouse_data
docker compose -f docker-compose.local.yml up -d clickhouse
dotnet run --project src/tools/DevDataSeeder -- --env local.env --skip-postgres
```

---

## 13. WBS Task Reference

This setup corresponds to **WBS task P0.11** in the project plan:

| ID | Task | Est. Days | Role(s) |
|---|---|---|---|
| P0.11 | Local dev environment: `docker-compose.local.yml`, `local.env.example`, dev AWS resource setup script, ClickHouse + Postgres init scripts, WireMock stub skeleton, `DevDataSeeder` CLI project scaffold | 2 | Architect + Dev 2 |

The `DevDataSeeder` implementation effort (fake opp generation, ClickHouse direct-write) is tracked separately as part of the Score Processor phase (P3) since it shares the field computation and normalization logic.
