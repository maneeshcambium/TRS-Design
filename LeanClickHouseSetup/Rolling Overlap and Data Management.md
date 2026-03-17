The confusion stems from the overlapping nature of academic years. To resolve this, the architecture uses a **Parallel Residency** strategy where the system supports two "Active" years in ClickHouse simultaneously before transitioning to the final Postgres snapshot.

### **1\. The Maintenance Timeline: "Rolling Overlap"**

Instead of a hard shutdown on a specific date, you maintain a "current" and a "trailing" instance. This ensures late scores for the previous year (e.g., 2025\) don't get lost while the new year (e.g., 2026\) is ramping up.

| Phase | Time Period (Approx.) | ClickHouse Status | Postgres Status |
| :---- | :---- | :---- | :---- |
| **Active Year N** | Aug 2025 – July 2026 | 2025 is the "Primary" instance. | CDC is streaming 2025 data. |
| **New Year N+1 Start** | Aug 2026 – Oct 2026 | **Parallel Period:** You run both 2025 and 2026 ClickHouse instances. | CDC streams both years to their respective instances. |
| **Snapshot Stage** | Nov 2026 | Perform the "Final Snapshot" of 2025 data into Postgres. | 2025 aggregates are moved to school\_overall\_aggregates. |
| **Sunset Stage** | Dec 2026 | **Shutdown 2025 ClickHouse.** Only 2026 remains active. | PeerDB mirror for 2025 is disabled. |

### ---

**2\. Managing trs.mv\_school\_standards\_current**

To answer your specific question: **No, this Materialized View (MV) should not hold both years' data all the time.**

Its purpose is to provide a high-performance fallback for the **active** scoring window only.

* **Year-Specific Definition:** When you create the MV, you define it with a hardcoded WHERE school\_year \= 2026 clause. This forces **partition pruning**, ensuring Postgres only scans the 2026 partition of the 1.6-billion-row component table.  
* **The Transition:** 1\. During the overlap (Aug–Oct), you might actually have **two** Materialized Views: mv\_standards\_2025 and mv\_standards\_2026.  
  2\. Once the "Snapshot" is complete for 2025, you **DROP** the mv\_standards\_2025.  
  3\. From that point forward, 2025 requests hit the **Snapshot Table** (Stage 3), and only 2026 requests hit the current MV.

### ---

**3\. Detailed Implementation of the Snapshot**

When you are ready to "clean" the instance, you follow this precise order to ensure no data loss:

1. **Execute Final Aggregation:** The Aggregate Refresh Lambda queries the ClickHouse Gold layer for the trailing year (2025) one last time.  
2. **Bulk Upsert to Postgres:** These final values are written into trs.school\_overall\_aggregates in the **ScoresDb**.  
3. **Update ConfigsDb:** Mark the year 2025 as "Archived" in your year\_registry table.  
4. **API Redirection:** The API Lambda sees the "Archived" status and stops attempting to route 2025 requests to ClickHouse, sending them to the Snapshot Table instead.  
5. **Infrastructure Cleanup:** Now—and only now—you stop the CDC mirror and delete the ClickHouse instance for that specific year.

### **Why this is "Clean"**

* **Indices stay small:** By dropping old MVs, the Postgres ScoresDb indices only track the active year, keeping lookups fast.  
* **No "Split Brain":** The year\_registry in **ConfigsDb** acts as the single source of truth for where the API should look for data (ClickHouse vs. Snapshot Table).  
* **Graceful Retirement:** Historical data remains queryable in a standard relational format without the cost of an EC2 instance.

**Would you like me to provide the SQL for the year\_registry table and the C\# Logic the API would use to route between "Active" and "Archived" years?**