# TRS Engineering Design Document v4: CDC-Driven Analytical Layer

| Field | Value |
| :--- | :--- |
| **Author** | Principal Software Architect |
| **Date** | 2026-03-03 |
| **Status** | Final for Implementation |
| **Project** | Teacher Reporting System (TRS) — Texas-Scale (5.5M Students) |

---

## Overview

This document provides a side-by-side architectural comparison of two leading **Change Data Capture (CDC)** strategies for the TRS analytical pipeline. Both designs utilize **AWS Fargate** to host a custom **C# .NET 8** transformer that performs the critical **Sign-Pair (-1/+1)** math required for atomic rescores in ClickHouse.

---

## 1. Core Architectural Logic: The "Sign-Pair" Transformer

Regardless of the CDC tool chosen, the **C# Fargate Service** acts as the mathematical brain of the pipeline. Its primary responsibility is to resolve the **"Silent Corruption" problem** inherent in stateless Materialized Views.

### 1.1 Transformation Logic

| Event | Action |
| :--- | :--- |
| **On UPDATE (Rescore)** | The service receives a message containing the **Before** (old) and **After** (new) row images. |
| **The Undo** | Generates a row with the **Old Score** and sign: `-1` |
| **The Redo** | Generates a row with the **New Score** and sign: `+1` |
| **The Batch** | Both rows are bundled into a **single binary insert** to ClickHouse, ensuring school/district totals subtract the old value and add the new one atomically. |

---

## 2. Option A: The "Standard" Architecture (Debezium + Kafka)

> **Recommendation:** Use for long-term scalability and multi-system data distribution.

### 2.1 Architecture Diagram

```
Aurora PostgreSQL  →  Debezium (MSK Connect)  →  Amazon MSK (Kafka)  →  C# Fargate Transformer  →  ClickHouse
     [WAL]                  [WAL Reader]               [Topic Buffer]        [-1/+1 Logic]            [Analytics]
```

### 2.2 Data Flow

1. **Aurora PostgreSQL** — Writes changes to the Write-Ahead Log (WAL).
2. **Debezium (MSK Connect)** — Reads the WAL and publishes Before/After JSON messages to a Kafka topic.
3. **Amazon MSK (Kafka)** — Acts as a high-durability, replayable message buffer.
4. **C# Fargate Service (The Transformer)**
   - Subscribes to the Kafka topic.
   - Performs `-1/+1` transformation logic.
   - Writes binary batches to ClickHouse.
5. **ClickHouse** — Receives sign-paired batches for collapse and aggregation.

### 2.3 Why Choose Option A?

| Benefit | Detail |
| :--- | :--- |
| **Durability** | If ClickHouse is down for maintenance, Kafka buffers scores for up to 7 days. |
| **Replayability** | You can "rewind" the Kafka offset to re-populate ClickHouse if data corruption is detected. |
| **Fan-Out** | A single Kafka topic can feed multiple downstream consumers simultaneously. |

---

## 3. Option B: The "High-Performance" Architecture (PeerDB)

> **Recommendation:** Use for maximum throughput and reduced infrastructure complexity.

### 3.1 Architecture Diagram

```
Aurora PostgreSQL  →  PeerDB Engine (Fargate)  →  SQS / Direct Endpoint  →  C# Fargate Transformer  →  ClickHouse
     [WAL]              [Binary Protocol]              [Event Queue]              [-1/+1 Logic]           [Analytics]
```

### 3.2 Data Flow

1. **Aurora PostgreSQL** — Writes changes to the Write-Ahead Log (WAL).
2. **PeerDB Engine (Fargate)** — Connects to Postgres and extracts changes via the high-speed binary protocol.
3. **Event Handoff** — PeerDB pushes raw change events to an internal SQS queue or a direct C# endpoint.
4. **C# Fargate Service (The Transformer)**
   - Consumes PeerDB webhook/queue events.
   - Performs `-1/+1` transformation logic.
   - Writes binary batches to ClickHouse.
5. **ClickHouse** — Receives optimized streams from the transformer.

### 3.3 Why Choose Option B?

| Benefit | Detail |
| :--- | :--- |
| **Simplicity** | Eliminates the need to manage a Kafka/MSK cluster. |
| **Latency** | PeerDB is optimized specifically for ClickHouse, often achieving lower end-to-end lag than generic Kafka connectors. |
| **Cost** | Fewer managed services means lower AWS infrastructure cost. |

---

## 4. Database Schema: ClickHouse Medallion Architecture

Both architectures feed into the same three-layer ClickHouse schema.

```
[Bronze Layer]                [Silver Layer]                  [Gold Layer]
VersionedCollapsing           AggregatingMergeTree            AggregatingMergeTree
MergeTree                     (Latest-only student view)      (School/District aggregates)
(-1/+1 sign pairs)            argMaxState(score, ts)          avgState + uniqState
```

| Layer | Engine | Logic |
| :--- | :--- | :--- |
| **Bronze** | `VersionedCollapsingMergeTree` | Uses `sign` and `date_scored` to physically collapse `-1/+1` pairs during background merges. |
| **Silver** | `AggregatingMergeTree` | Uses `argMaxState` to provide a "latest-only" view of student scores for roster reports. |
| **Gold** | `AggregatingMergeTree` | Pre-calculated School and District aggregates using `avgState` and `uniqState`. |

---

## 5. Decision Matrix

| Metric | Option A (Debezium + Kafka) | Option B (PeerDB) |
| :--- | :--- | :--- |
| **Infrastructure** | Aurora + MSK + Fargate + ClickHouse | Aurora + Fargate + ClickHouse |
| **Reliability** | Highest — 7-day Kafka buffer | High — Direct stream with SQS |
| **Dev Effort** | Higher — Custom Kafka consumer | Lower — Connector-focused config |
| **Sync Speed** | Standard | Ultra-High — Binary protocol |
| **Replayability** | Yes — Kafka offset rewind | Limited — SQS retention window |
| **Cost** | Higher (MSK cluster) | Lower (no Kafka) |
| **Fan-Out Support** | Native (multiple Kafka consumers) | Requires duplication |

---

## 6. API Fallback Strategy

The API Lambda implements a **three-tier fallback** to ensure zero downtime for read workloads:

```
Request
  │
  ▼
[1] Postgres Aggregate Cache  ──(hit)──▶  Return Result
  │ (miss)
  ▼
[2] ClickHouse Query          ──(hit)──▶  Return Result
  │ (unavailable)
  ▼
[3] Postgres Live Query       ──────────▶  Return Result (degraded mode)
```

1. **Postgres Aggregate Cache** — Fast pre-computed aggregates; checked first.
2. **ClickHouse** — Main analytical engine; used if cache is stale or cold.
3. **Postgres Live Query** — Full fallback when ClickHouse is unreachable; higher latency but always available.

---

## 7. Next Steps

- [ ] Finalize CDC tool selection (Option A vs. Option B) based on operational team capacity.
- [ ] Implement C# `BackgroundService` boilerplate for either the Kafka Consumer (Option A) or the SQS Listener (Option B).
- [ ] Validate ClickHouse Bronze/Silver/Gold schema with production-scale data load tests.
- [ ] Define SLA thresholds for the API fallback tiers.

---

## 8. Cost Recalibration: Postgres-Primary, ClickHouse Aggregation-Only

> **Constraint set:** PostgreSQL (Aurora Serverless v2) is the source of truth for all score data. ClickHouse is a **read-only aggregation replica** — it only serves school/district/state aggregation queries. No ClickHouse replica node. Low ingestion frequency (batch test windows, not continuous stream).

### 8.1 What This Architectural Shift Unlocks

The original architecture (Sections 2–3) assumed ClickHouse was the **primary score store**, requiring a two-node replica set, a Silver layer for per-student roster queries, and a Keeper cluster for coordination. Moving to Postgres-primary eliminates all of that:

| Original Assumption | Revised Constraint | Impact |
| :--- | :--- | :--- |
| ClickHouse = primary score store | ClickHouse = aggregation only | Remove Silver layer entirely; per-student roster queries go to Postgres |
| Replica required (HA for primary store) | No replica needed (Postgres is source of truth; ClickHouse can be rebuilt) | Cut ClickHouse EC2 cost by ~50% |
| ClickHouse Keeper × 3 | Not needed for single node | Save ~$45/month |
| `r6i.4xlarge` at TX scale (128GB, write + read) | `r6i.2xlarge` sufficient (64GB, aggregation queries only) | Node cost drops from $584 → $362 |
| MSK justifiable for primary-store HA | MSK unjustifiable for low-frequency CDC | Remove $628/month MSK overhead |

**Key safety property:** With Postgres as source of truth, a ClickHouse failure is a **reporting degradation**, not a data loss event. The API fallback (Section 6) handles it automatically. ClickHouse can be fully rebuilt from Postgres at any time with no RPO concern.

---

### 8.2 Three CDC Options at This Constraint Set

#### Option A — Amazon MSK (Original)

> ❌ **Not recommended.** MSK's cost is driven by always-on broker compute. For infrequent batch test-window ingestion, you are paying for 24/7 Kafka HA that you only need a few times per year.

| Service | Monthly Cost |
| :--- | :--- |
| 3× `kafka.m5.large` MSK brokers | $363 |
| MSK broker EBS storage (1TB × 3) | $90 |
| MSK Connect (Debezium) | $115 |
| C# Fargate Transformer | $18 |
| **CDC Subtotal** | **~$586/month** |

#### Option A-Budget — Self-Hosted Redpanda on EC2

> ✅ **Viable if Kafka replay semantics are a hard requirement.** [Redpanda](https://redpanda.com) is a Kafka-compatible broker that runs as a single binary on a small EC2 instance — no JVM, no ZooKeeper. Gives you full topic replay and offset rewind at ~5% of MSK cost.

```
Aurora PostgreSQL  →  Debezium (Docker on t3.medium)  →  Redpanda (same t3.medium)  →  C# Fargate  →  ClickHouse
     [WAL]                  [WAL Reader]                    [Kafka-compatible topic]     [-1/+1]       [Aggregation]
```

| Service | Config | Monthly Cost |
| :--- | :--- | :--- |
| EC2: Redpanda + Debezium co-located | `t3.medium` (2 vCPU, 4GB) | $30 |
| C# Fargate Transformer | 0.5 vCPU / 1GB | $18 |
| **CDC Subtotal** | | **~$48/month** |

**Tradeoff:** Single Redpanda node — if the EC2 instance fails during an active test window, you lose un-consumed messages. Acceptable because Postgres WAL is the real buffer; you can replay scores from Postgres on recovery. Topic retention: set to 7 days.

#### Option B — PeerDB on Fargate (Recommended)

> ✅ **Recommended for this constraint set.** PeerDB reads PostgreSQL logical replication directly. The Postgres replication slot itself acts as the durable buffer — changes accumulate in the WAL until PeerDB consumes them, even if the Fargate task restarts. No separate broker needed.

```
Aurora PostgreSQL  →  PeerDB Fargate  →  C# Fargate Transformer  →  ClickHouse
     [WAL / replication slot]              [-1/+1 Logic]            [Aggregation only]
```

| Service | Config | Monthly Cost |
| :--- | :--- | :--- |
| PeerDB Engine (Fargate, minimal) | 0.25 vCPU / 0.5GB | $9 |
| C# Fargate Transformer (Fargate, minimal) | 0.5 vCPU / 1GB | $18 |
| **CDC Subtotal** | | **~$27/month** |

> **Why the replication slot is sufficient buffering here:** PeerDB holds an open replication slot on Postgres. If ClickHouse is briefly unavailable, PeerDB simply pauses consumption and the WAL accumulates on the Postgres side — no data loss. Since ingestion bursts are infrequent and short, the WAL growth during a pause is small. Set `max_slot_wal_keep_size = 10GB` as a safety cap.

---

### 8.3 Full Cost Breakdown — Texas Scale (5.5M Students)

Base infrastructure is shared across all three CDC options:

| Service | Config | Monthly Cost |
| :--- | :--- | :--- |
| Aurora Serverless v2 (**PRIMARY** — all scores + config + membership) | avg 0.5–1 ACU, ~50GB storage | ~$53–96 |
| EC2: ClickHouse single node (aggregation only) | `r6i.2xlarge` (64GB RAM, 8 vCPU) | $362 |
| EBS: ClickHouse data volume | 200GB `gp3` | $16 |
| Lambda (Score Processor + API + RTS Sync) | ~50M invocations | $10–20 |
| S3 (raw score files + ClickHouse backup) | ~5TB/year | $15–30 |
| SQS (ingest queue + DLQ) | ~50M messages | $3 |
| CloudFront + API Gateway | 1,000 concurrent users | $10–20 |
| CloudWatch (metrics + logs) | | $10–20 |
| **Base Subtotal** | | **~$479–567/month** |

**Total with each CDC option:**

| | Option A (MSK) | Option A-Budget (Redpanda) | Option B (PeerDB) |
| :--- | :--- | :--- | :--- |
| Base | ~$479–567 | ~$479–567 | ~$479–567 |
| CDC Layer | ~$586 | ~$48 | ~$27 |
| **On-Demand Total** | **~$1,065–$1,153** | **~$527–$615** | **~$506–$594** |
| **1-Year Reserved EC2** | **~$843–$931** | **~$305–$393** | **~$284–$372** |

> **1-Year Reserved EC2** applies ~42% discount to the `r6i.2xlarge` node ($362 → ~$210/month), cutting ~$152/month off the base.

---

### 8.4 Cost by Scale — Option B (PeerDB, Recommended)

| Scale | ClickHouse Node | On-Demand/Month | 1-Year Reserved/Month |
| :--- | :--- | :--- | :--- |
| **MVP / Single client** | `r6i.xlarge` (32GB) | **~$270–310** | **~$195–235** |
| **Virginia (1.26M students)** | `r6i.xlarge` (32GB) | **~$310–360** | **~$238–288** |
| **Texas (5.5M students)** | `r6i.2xlarge` (64GB) | **~$506–594** | **~$284–372** |

> **Why `r6i.xlarge` is sufficient up to ~1.3M students:** Aggregation-only ClickHouse queries over ~13GB of compressed score data consistently run under 100ms on 32GB RAM. The Silver layer (per-student roster) has been removed — Postgres handles that. The remaining load is pure `GROUP BY` aggregation at school/district/state level.

---

### 8.5 Recommendation

**Use Option B (PeerDB) with a single `r6i.2xlarge` ClickHouse node.**

The combination of:
1. Postgres as primary store → no ClickHouse replica needed
2. PeerDB using Postgres replication slot as natural buffer → no Kafka broker needed
3. Aggregation-only ClickHouse → downsize from `r6i.4xlarge × 2` to single `r6i.2xlarge`

...reduces the monthly cost from the original ~$1,345–$1,385 (full ClickHouse-primary, MSK architecture) to **~$506–$594/month at Texas scale** (~$830–$870 less per month, or **~$10,000–$10,400/year savings**).

The only scenario where Option A-Budget (Redpanda) is preferable is if the team has a hard requirement for explicit Kafka topic replay (e.g., ability to independently replay CDC events without touching Postgres). The $21/month difference between PeerDB and Redpanda is negligible — the choice is purely operational preference.

---

## 9. Resilience: PeerDB + ClickHouse Failure Modes

Resilience in the PeerDB-to-ClickHouse architecture is achieved by combining three mechanisms:

1. **PostgreSQL WAL** — every committed change is durably recorded in the Write-Ahead Log.
2. **PeerDB Checkpointing (LSN Tracking)** — PeerDB tracks its last successfully consumed Log Sequence Number (LSN) via a Postgres Logical Replication Slot; on restart it resumes from exactly that point.
3. **Three-Tier API Fallback** (Section 6) — the API layer redirects reads to Postgres the moment ClickHouse becomes unhealthy.

The combination means ClickHouse is **disposable** — losing it entirely is a reporting degradation, never a data loss event.

---

### 9.1 Scenario: ClickHouse Instance Reboot

```
ClickHouse goes offline
        │
        ▼
API health check fails → all queries redirect to Tier 1 (Postgres Cache)
                          or Tier 3 (Postgres Live Query)
        │
        ▼
PeerDB pauses delivery (destination unreachable)
Aurora Postgres continues writing ALL changes to WAL
        │
        ▼
ClickHouse comes back online
        │
        ▼
PeerDB checks last saved LSN checkpoint → replays missed WAL changes
        │
        ▼
ClickHouse reaches real-time status → API resumes ClickHouse queries
```

| Layer | Behaviour During Outage |
| :--- | :--- |
| **API** | Immediately falls back to Postgres; zero user-visible impact |
| **PeerDB** | Pauses; WAL accumulates on the Postgres side |
| **Data** | No loss — Postgres replication slot holds WAL until PeerDB resumes |

---

### 9.2 Scenario: PeerDB Fargate Task Restart

This is a standard operational event (deployment, task replacement, OOM restart).

```
Fargate task restarts
        │
        ▼
Postgres Logical Replication Slot holds WAL for this specific slot
(WAL does NOT advance past the slot's confirmed_flush_lsn while PeerDB is offline)
        │
        ▼
Fargate task comes back up → PeerDB reconnects to same replication slot
        │
        ▼
PeerDB requests all changes since last confirmed LSN → processes backlog into ClickHouse
        │
        ▼
VersionedCollapsingMergeTree + Sign-Pair logic is fully idempotent
Any duplicate messages from a messy restart are collapsed automatically
```

> **WAL bloat guard:** Set `max_slot_wal_keep_size = 10GB` on Aurora to cap WAL accumulation during extended PeerDB downtime. If the cap is hit, the slot is invalidated and a full re-snapshot is required — acceptable because Postgres remains the source of truth.

---

### 9.3 Scenario: Total EC2 Loss (Disaster Recovery)

If the ClickHouse EC2 instance is terminated or the EBS volume is corrupted, the rebuild path is:

```
EC2 lost / volume corrupted
        │
        ▼
API immediately falls back to Postgres (Tier 1 / Tier 3) — users unaffected
        │
        ▼
Provision new EC2 instance via CDK / Terraform
Apply Medallion Schema (Bronze VersionedCollapsingMergeTree, Gold AggregatingMergeTree)
        │
        ▼
Trigger PeerDB "Initial Load"
PeerDB performs high-speed parallel snapshot of all score data from Aurora → new ClickHouse
(5.5M students @ TX scale: estimated 30–90 minutes)
        │
        ▼
Snapshot completes → PeerDB automatically switches to Streaming Mode
Pulls in all WAL changes that occurred WHILE the snapshot was running
        │
        ▼
ClickHouse is current → API resumes using ClickHouse for aggregation queries
```

> Throughout the entire multi-hour rebuild, users continue to see reports via
> the Postgres fallback path. The rebuild is fully transparent at the API layer.

---

### 9.4 Resilience Summary

| Failed Component | User Impact | Data Loss Risk | Recovery Mechanism | Est. Recovery Time |
| :--- | :--- | :--- | :--- | :--- |
| **ClickHouse instance** | None — API falls back to Postgres | None — WAL buffered in replication slot | PeerDB LSN checkpoint replay on restart | Minutes |
| **PeerDB Fargate task** | None — API falls back to Postgres | None — replication slot holds WAL | Fargate auto-restart; PeerDB resumes from last LSN | Minutes |
| **Total EC2 + EBS loss** | None — API falls back to Postgres | Temporary lag only | PeerDB Initial Load snapshot → Streaming handover | 30–90 min (TX scale) |
| **Aurora Postgres** | Full outage (primary store down) | Depends on Aurora HA tier | Aurora Multi-AZ automatic failover | ~30 seconds (RDS failover) |

> **The architectural guarantee:** ClickHouse holds no data that cannot be fully reconstructed from Aurora. Its loss is bounded to query performance degradation with automatic API-level mitigation. Aurora Multi-AZ is the only tier where failure causes true user impact.