Yes, exactly. During the "rollover" months (typically August through December), you would run **two separate ClickHouse instances** (e.g., two small EC2 nodes or two separate ClickHouse Cloud services).

This is the "secret sauce" for a high-performance system at the Texas scale. Instead of asking one database to be both a real-time ingestion engine for 2026 and a historical analysis engine for 2025, you isolate the workloads. This allows you to **shut down the 2025 compute entirely** once the data stabilizes, dropping your costs to near-zero for that year while keeping the aggregates safely in Postgres.

# ---

**Teacher Reporting System (TRS) — Engineering Design Document v5**

**Author:** Principal Software Architect

**Date:** March 4, 2026

**Status:** Final Specification — Implementation Phase

## **1\. Multi-DB Strategy: Functional Isolation**

The system is split into two specialized PostgreSQL instances to decouple metadata management from high-volume transaction processing.

### **1.1 ConfigsDb (The Brain)**

* **Role:** Stores the "Rules of the World."  
* **Tables:** tenants, test\_configs (measures, PL counts), year\_registry (defines which years are "Active" vs. "Archived"), and embargo\_rules.  
* **Traffic:** Low volume, high criticality.

### **1.2 ScoresDb (The Muscle)**

* **Role:** Stores all score data and membership relationships.  
* **Tables:** student\_opportunities, student\_component\_scores, school\_student (membership), and the **Snapshot Table** school\_overall\_aggregates.  
* **Traffic:** High volume; target of all CDC streaming.

## ---

**2\. The Data Lifecycle: "Snapshot & Sunset"**

Every school year moves through a controlled lifecycle to ensure the "Current Year" always runs on the leanest possible infrastructure.

### **Stage 1: Active Ingestion (e.g., 2026\)**

* **Infrastructure:** 2026 ClickHouse Instance \+ PeerDB Mirror (Filter: year=2026).  
* **Path:** API → report\_cache → ClickHouse 2026\.

### **Stage 2: The Parallel Overlap (Aug – Dec 2026\)**

* **Scenario:** 2026 testing begins, but 2025 rescores are still trickling in.  
* **Architecture:** \* **2026 ClickHouse Instance:** Busy, high-tier compute node.  
  * **2025 ClickHouse Instance:** Downsized, low-tier compute node (e.g., t4g.medium).  
* **Routing:** API Lambda checks year\_registry. If year is 2025, it calls CH-2025; if 2026, it calls CH-2026.

### **Stage 3: Snapshot & Shutdown (Dec 2026\)**

* **Action:** The **Aggregate Refresh Lambda** performs a "Final Sweep".  
* **Logic:** It reads all 2025 Gold aggregates from ClickHouse and UPSERTs them into the relational school\_overall\_aggregates table in **ScoresDb**.  
* **Cleanup:** 1\. Update year\_registry (2025 \= Archived).  
  2\. Stop PeerDB 2025 Mirror.  
  3\. **Terminate 2025 ClickHouse Instance.**

## ---

**3\. Tiered API Routing Logic (The Bridge)**

The API Lambda uses the year\_registry to determine the most cost-effective path for a request.

| Requested Year | Status | Source Path |
| :---- | :---- | :---- |
| **2026** | Active | **T1:** report\_cache → **T2:** ClickHouse 2026 → **T3:** Postgres MV. |
| **2025** | Active | Same as above, but hitting ClickHouse 2025\. |
| **Historical** | Archived | **T1:** report\_cache → **T3:** Postgres Snapshot Table (school\_overall\_aggregates). |

## ---

**4\. Key Engineering Details**

### **4.1 The Snapshot Table Schema**

Designed for permanent, relational storage of billions of aggregate rows without needing ClickHouse for retrieval.

* **Columns:** tenant\_id, school\_year, school\_id, test\_group\_id, students\_tested, avg\_scale\_score, pl\_1\_count... pl\_10\_count.  
* **Indexing:** Partitioned by tenant\_id (LIST) with a primary key on (school\_year, test\_group\_id, school\_id).

### **4.2 Component Score Fallback**

For the 1.6-billion-row standards table (student\_component\_scores), we **do not** snapshot every possible aggregate (too many combinations).

* **Active Years:** Use ClickHouse.  
* **Archived Years:** If a standard report is requested for 2024, the API performs a **Live Query** on the Postgres partition (fast due to pruning) and caches it in report\_cache with a 7-day TTL.

## ---

**5\. Maintenance Summary**

| Activity | Frequency | Impact |
| :---- | :---- | :---- |
| **Provision New CH Instance** | Annual (August) | Low — Spin up small EC2, run DDL scripts. |
| **Final Snapshot Job** | Annual (December) | Medium — One-time bulk move of data to ScoresDb. |
| **Decommission CH Instance** | Annual (December) | High — Immediate 50% reduction in ClickHouse bill. |

---

**Would you like me to generate the C\# YearRegistryService and ConnectionFactory code that implements this "Active vs. Archived" routing?**