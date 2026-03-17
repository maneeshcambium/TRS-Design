Gemini said
You've spotted the most critical distinction in the engine choice. While the Relationship Mirrors use ReplacingMergeTree to maintain the "Current State" (latest wins), the Scores Bronze table uses VersionedCollapsingMergeTree to maintain a "Mathematical Ledger".

Here is why ReplacingMergeTree would actually break your aggregations, and why VersionedCollapsing is the only way to handle 1.6 billion rows of shifting score data.

1. ReplacingMT "Erases" History; VersionedCollapsing "Balances" It
ReplacingMergeTree (Relationships): This engine is designed to delete the old row when a new version arrives. If a student moves from School A to School B, the record for School A is eventually physically deleted.

VersionedCollapsingMergeTree (Scores): This engine is designed to keep rows that represent "cancellation." It uses the sign column to tell ClickHouse: "This row isn't just an update; it's a mathematical instruction to undo a previous entry".

2. The Aggregation "Snapshot" Problem
The Materialized View (MV) is the main reason for this choice.

With ReplacingMT: When an update happens (85 → 90), the engine eventually replaces the 85 with 90. But the MV triggers the moment the row is inserted. If the engine hasn't "replaced" the row yet, the MV might see both rows as positive values, causing your school average to spike incorrectly.

With VersionedCollapsing: When the -1 (85) and the +1 (90) arrive, the MV processes them mathematically.

Sum = (Current Total) + (-85) + (90).

The result is immediate and perfectly accurate, even if the background merge hasn't happened yet.

3. The "Rescore" Resilience
In a Texas-scale rescore, you might have 5 million students whose scores change simultaneously.

ReplacingMT would struggle with the massive "version" conflict resolution during such high-volume updates.

VersionedCollapsing loves this. It just appends the -1/+1 pairs to the end of the data parts. It doesn't have to "find" the old row to replace it. This is why ClickHouse can ingest millions of rows per second—it's just adding to a ledger, not searching an index.

Comparison: Which engine for which task?
Feature	ReplacingMergeTree (Relationships)	VersionedCollapsingMergeTree (Scores)
Logic	"Give me the latest version."	"Sum the +1 and -1 ledger."
Storage	Keeps only the winner.	Keeps pairs that cancel out until merge.
MV Accuracy	Risky (can overcount before merge).	Perfect (Sign-pairs are atomic math).
Best For	Slow-moving dimensions (School, Grade).	Fast-moving metrics (Scores, PLs).