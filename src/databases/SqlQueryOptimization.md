# SQL Query Optimization — A Practical & Interview-Ready Guide

*Anchored to your resume line: "Optimised PostgreSQL queries for a bulk emailing platform through query restructuring and indexing, reducing recipient retrieval time by 95% and accelerating campaign execution."*

This guide explains, from first principles, **how queries actually run**, **why they get slow**, **how to make them fast**, and **how to prove the improvement with numbers**. Examples are PostgreSQL-first (your Epikso work) with MySQL notes where they differ, since both are on your resume.

---

## 0. The mental model: how a query becomes work

When you send SQL, the database does not "run your text." It runs a **plan**. Understanding this pipeline is the single most useful thing for talking about optimization:

1. **Parser** — checks syntax, turns SQL into a tree.
2. **Rewriter** — applies view definitions, rules.
3. **Planner / Optimizer** — this is the important one. It considers *many* possible ways to execute the query (which index to use, which join order, which join algorithm) and estimates the **cost** of each using table statistics. It picks the cheapest.
4. **Executor** — runs the chosen plan and returns rows.

Key insight: **the same SQL can run in wildly different ways.** "Optimizing a query" usually means *changing the plan the optimizer picks* — by giving it a better index, rewriting the query so a good plan becomes possible, or updating the statistics it reasons with.

Two costs dominate almost every slow query:

- **Rows read from disk/memory** — how much data the DB has to touch. A *sequential scan* reads the whole table; an *index scan* jumps to just the rows you need.
- **Work per row** — sorting, hashing, joining, function evaluation.

Optimization is mostly about **touching fewer rows** and **doing less work per row**.

---

## 1. How to tell if a query is slow (detection)

You cannot optimize what you cannot see. There are three levels of visibility.

### 1.1 Time a single query
The crudest check — run it and look at the wall-clock time.

```sql
-- psql
\timing on
SELECT ...;
```

Useful but misleading: it includes network + client render time, and a single run is noisy (caching effects). Use it only for a rough smell test.

### 1.2 Find the slow queries across the whole system
The real question in production is "*which* of my thousands of queries are the problem?" You almost never know upfront. Two mechanisms:

**A. `pg_stat_statements` (PostgreSQL — the most important tool here).**
It's an extension that aggregates execution stats per *normalized* query (literals stripped, so `WHERE id = 1` and `WHERE id = 2` group together).

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 queries by TOTAL time spent (the real resource hogs)
SELECT query,
       calls,
       total_exec_time,
       mean_exec_time,
       rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

Why **total** time and not just mean? A query that takes 5 ms but runs 10 million times/day hurts more than a 2-second query run twice. Optimizing the former gives you the bigger win. This "total time = mean × calls" framing is a strong interview point.

**B. The slow query log.**
Log any statement over a threshold, then analyze offline.

```ini
# postgresql.conf
log_min_duration_statement = 500   # log statements slower than 500 ms
```

MySQL equivalent:
```ini
slow_query_log = 1
long_query_time = 0.5              # seconds
```
Then aggregate with `pt-query-digest` (Percona Toolkit) or `mysqldumpslow`.

### 1.3 Symptoms that point at query problems
- Latency rises with **data volume** or **concurrency** but CPU/RAM aren't maxed → likely full scans / missing indexes.
- A query is fast on a small dev DB, slow in prod → data-size-dependent plan (seq scan tipping point), or stale statistics.
- Latency spikes are periodic → lock contention or a batch job competing for I/O.

---

## 2. Reading the plan: EXPLAIN and EXPLAIN ANALYZE

This is the heart of diagnosis and the thing interviewers probe hardest.

- **`EXPLAIN`** shows the plan the optimizer *intends* to use, with *estimated* costs. It does **not** run the query.
- **`EXPLAIN ANALYZE`** actually **runs** the query and shows *real* timings and *actual* row counts alongside the estimates. (Caution: it executes the statement, so wrap writes in a transaction you roll back.)

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT r.email
FROM recipients r
WHERE r.campaign_id = 42
  AND r.status = 'PENDING';
```

### 2.1 What to read in the output
Example (annotated):

```
Seq Scan on recipients r  (cost=0.00..21850.00 rows=980 width=32)
                          (actual time=0.02..142.11 rows=970 loops=1)
  Filter: ((campaign_id = 42) AND (status = 'PENDING'))
  Rows Removed by Filter: 1999030
Planning Time: 0.15 ms
Execution Time: 143.02 ms
```

How to interpret, line by line:

- **`Seq Scan`** — it read the *entire* table. Red flag when you're selecting a small slice. You want an `Index Scan` here.
- **`cost=0.00..21850.00`** — planner's estimate in arbitrary units: *startup cost*..*total cost*. Compare costs *between* plans, don't treat the absolute number as milliseconds.
- **`rows=980`** (estimate) vs **`actual ... rows=970`** — these should be close. A big mismatch (e.g. estimate 980, actual 900,000) means **stale/insufficient statistics** and is the root cause of many bad plans. Fix with `ANALYZE`.
- **`Rows Removed by Filter: 1999030`** — the smoking gun. It touched ~2M rows to return 970. That work is exactly what an index eliminates.
- **`BUFFERS`** (when enabled) — shows `shared hit` (from cache) vs `read` (from disk). Lots of `read` = I/O-bound; more caching or a tighter index helps.
- **`Execution Time`** — the real number to track before/after.

### 2.2 The scan/join vocabulary to know
Scan types (cheapest intent to most expensive):
- **Index Only Scan** — answer comes entirely from the index, table never touched. Fastest.
- **Index Scan** — index finds row locations, then fetches rows from the table.
- **Bitmap Index Scan** — builds a bitmap of matching rows first; good when a medium number of rows match.
- **Seq Scan** — read everything. Correct choice when you actually need most of the table; a problem when you need a few rows.

Join algorithms:
- **Nested Loop** — for each row on the left, look up matches on the right. Great when one side is tiny (and the right side is indexed); terrible for two large unindexed sets.
- **Hash Join** — build a hash table of one side, probe with the other. Good for large, unsorted sets with equality joins.
- **Merge Join** — both sides sorted, then merged. Good when inputs are already sorted (e.g., by index).

When you see a slow **Nested Loop** over big tables, that's often a missing join index.

MySQL note: use `EXPLAIN` and `EXPLAIN ANALYZE` (8.0+); the `type` column is the key signal — `ALL` = full scan (bad), `ref`/`range` = index use, `const`/`eq_ref` = best. `EXPLAIN FORMAT=JSON` gives cost detail.

---

## 3. Indexing — the highest-leverage lever

An index is a separate, sorted data structure (usually a **B-tree**) that lets the DB find rows without scanning the table — the same reason a book index beats reading every page. This is almost certainly the core of your "reducing recipient retrieval time by 95%" result.

### 3.1 When an index helps
- Columns in `WHERE`, `JOIN ... ON`, and `ORDER BY`.
- **High selectivity** columns — ones that narrow to few rows (e.g., `email`, `campaign_id`). Indexing a low-cardinality column like a boolean `is_active` rarely helps because the DB still reads most of the table.

```sql
CREATE INDEX idx_recipients_campaign_status
ON recipients (campaign_id, status);
```

### 3.2 Composite (multi-column) indexes and column order
Order matters. A B-tree on `(campaign_id, status)` supports:
- `WHERE campaign_id = 42` ✓
- `WHERE campaign_id = 42 AND status = 'PENDING'` ✓
- `WHERE status = 'PENDING'` alone ✗ (can't skip the leading column efficiently)

**Rule of thumb (leftmost prefix):** put the column used for equality first, and the most selective / most-frequently-filtered column early. This is a classic interview question — be ready to explain why `WHERE status = 'PENDING'` alone can't use the index above.

### 3.3 Covering / Index-Only Scans
If the index contains *every column the query needs*, the DB never visits the table — an **Index Only Scan**.

```sql
-- Postgres: include extra columns so SELECT email is covered
CREATE INDEX idx_recipients_lookup
ON recipients (campaign_id, status) INCLUDE (email);
```
MySQL: put the columns in the composite index itself; InnoDB indexes always carry the primary key.

### 3.4 Specialized index types (Postgres)
- **B-tree** — default; equality and range (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`).
- **Hash** — equality only; rarely worth it over B-tree.
- **GIN** — "one row, many values": full-text search, `jsonb`, array containment.
- **GiST / SP-GiST** — geometric, ranges, nearest-neighbor.
- **BRIN** — tiny index for huge, naturally-ordered tables (e.g., append-only time-series); stores min/max per block.
- **Partial index** — index only the rows you query:
  ```sql
  CREATE INDEX idx_pending ON recipients (campaign_id)
  WHERE status = 'PENDING';
  ```
  Smaller and faster when you always filter on that condition — perfect for a "send the pending ones" workload.
- **Expression index** — index the *result* of a function so the query can use it:
  ```sql
  CREATE INDEX idx_lower_email ON recipients (LOWER(email));
  -- now WHERE LOWER(email) = '...' is indexable
  ```

### 3.5 The cost of indexes (don't over-index)
Indexes are not free:
- Every `INSERT`/`UPDATE`/`DELETE` must update every affected index → **writes get slower**.
- They consume disk and RAM (cache pressure).
- Redundant indexes (e.g., `(a)` when `(a,b)` exists) waste resources.

The engineering judgment — "I added the *minimum* indexes that covered the hot read paths without hurting write throughput" — is exactly the nuance that makes the resume line credible.

Find unused indexes in Postgres:
```sql
SELECT indexrelid::regclass AS index, idx_scan AS times_used
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;   -- idx_scan = 0 → candidate for removal
```

---

## 4. Query restructuring — making a better plan possible

The other half of your resume line. Indexes help only if the query is written so the DB *can* use them. Common rewrites:

### 4.1 Avoid functions on indexed columns ("sargability")
A predicate is **SARGable** (Search-ARGument-able) if the DB can use an index for it. Wrapping the column in a function usually kills index use:

```sql
-- NOT sargable: index on created_at is ignored
WHERE DATE(created_at) = '2024-01-01'

-- Sargable: range on the raw column, index usable
WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02'
```
Same problem with `WHERE amount * 1.1 > 100` (move the math to the constant side) and leading-wildcard `LIKE '%foo'` (can't use a normal B-tree; needs a trigram/GIN index).

### 4.2 Select only what you need
`SELECT *` fetches unneeded columns, defeats index-only scans, and bloats network payload. Name the columns. Similarly, always **paginate** large result sets (keyset pagination beats large `OFFSET`, which still scans and discards).

```sql
-- Keyset pagination: fast at any depth
WHERE (campaign_id, id) > (42, :last_id)
ORDER BY campaign_id, id
LIMIT 100;
```

### 4.3 Prefer `EXISTS` / `JOIN` over correlated subqueries and `IN (SELECT ...)`
```sql
-- Often slow: re-evaluated per row
WHERE user_id IN (SELECT user_id FROM unsubscribes)

-- Usually better: set-based, hash-joinable
WHERE NOT EXISTS (
  SELECT 1 FROM unsubscribes u WHERE u.user_id = r.user_id
)
```

### 4.4 Batch instead of row-by-row (the N+1 problem)
The classic app-layer killer: a loop issuing one query per iteration (common with ORMs like Hibernate, which you use). 10,000 recipients → 10,000 round-trips. Replace with a single set-based query or `JOIN FETCH` / batch fetching. For a bulk emailing platform this is frequently the biggest single win — worth calling out explicitly.

### 4.5 Reduce work in joins and aggregates
- Filter **before** joining/aggregating so fewer rows flow through.
- Make sure both sides of a join key are the **same type** (an `int` vs `bigint`/`text` mismatch can force a cast that disables the index).
- Consider **materialized views** for expensive aggregations that don't need to be real-time:
  ```sql
  CREATE MATERIALIZED VIEW campaign_stats AS SELECT ...;
  REFRESH MATERIALIZED VIEW CONCURRENTLY campaign_stats;
  ```

### 4.6 Keep statistics fresh
The optimizer's decisions are only as good as its stats. After big data changes, run `ANALYZE`. Autovacuum usually handles this, but bulk loads can outrun it, producing the estimate-vs-actual mismatches from §2.1.

```sql
ANALYZE recipients;                       -- refresh stats
-- Increase detail on a skewed column:
ALTER TABLE recipients ALTER COLUMN status SET STATISTICS 1000;
```

Also relevant: **VACUUM** reclaims space from dead rows (Postgres's MVCC leaves them behind); table **bloat** silently slows scans over time.

---

## 5. Measuring before/after — proving the "95%"

An optimization you can't measure is a guess. Here's a rigorous, defensible methodology — the kind of answer that impresses in interviews.

### 5.1 Establish a clean baseline
1. Pick a **representative query** with **representative data volume** (prod-like, not a 100-row dev table — plans change with size).
2. **Warm vs cold cache:** the first run reads from disk, later runs hit cache. Decide which you're measuring and be consistent. Run several times, discard the first, take the **median** (not the mean — it's robust to outliers).
3. Capture the **plan** (`EXPLAIN (ANALYZE, BUFFERS)`), not just the time — the plan explains *why* it changed and protects you against "it was just cache."
4. Record: median execution time, rows read (`Rows Removed by Filter`, buffer hits/reads), and the scan/join types.

### 5.2 Make one change at a time
Change the index *or* the rewrite, not both at once, so you know what caused the improvement. Re-run the identical measurement.

### 5.3 Compute and express the delta
- **% reduction in latency** = `(before − after) / before × 100`.
    - 2000 ms → 100 ms is a **95%** reduction (and a **20×** speedup). Note: "95% faster" and "20× faster" describe the *same* change — know both framings.
- Also cite the *mechanism*: "seq scan over 2M rows → index scan touching ~1k rows," and rows-read dropping by orders of magnitude. Latency + mechanism together is far more convincing than a bare percentage.

### 5.4 Verify under realistic load, not just single-shot
Single-query timing hides concurrency effects. Use a load tool to confirm the win holds at scale (you already list **JMeter**):
- **JMeter** — model concurrent users/threads hitting the API or DB; compare p50/p95/p99 latency and throughput before vs after.
- **pgbench** (Postgres) — scriptable benchmark for TPS and latency percentiles.
- Track **percentiles (p95/p99)**, not just averages — averages hide tail latency, which is what users actually feel. This distinction is a common senior-level interview point.

### 5.5 Confirm the win in production
After deploying, watch `pg_stat_statements` (`mean_exec_time`, `total_exec_time` for that query) and your dashboards trend down. That closes the loop from "faster in a benchmark" to "faster for real traffic."

---

## 6. Monitoring — catching slowness before users do

Optimization isn't a one-off; data grows and plans drift. Monitoring turns it into a continuous practice. Your stack already has the right pieces: **Prometheus + Grafana**.

### 6.1 What monitoring buys you
- **Detection:** alert when p95 latency or slow-query count crosses a threshold.
- **Trend/regression:** see a query degrade as data grows (the seq-scan tipping point) *before* it's an incident.
- **Prioritization:** dashboards rank the worst offenders so you optimize what matters.
- **Correlation:** tie a latency spike to a deploy, a batch job, a lock storm, or a traffic surge.

### 6.2 The metrics that matter
- **Query latency percentiles** (p50/p95/p99) per endpoint/query.
- **Throughput** (queries/sec, transactions/sec).
- **Cache hit ratio** — Postgres buffer cache hit rate; a falling ratio means more disk I/O.
- **Locks / blocked queries / deadlocks** — contention, not query shape, causes many "random" slowdowns.
- **Connections** — saturating the pool queues everything; often a case for **PgBouncer** connection pooling.
- **Replication lag**, **table/index bloat**, **long-running transactions** (they block autovacuum).
- **Host signals:** CPU, memory, disk IOPS/latency.

### 6.3 The tool chain (maps to your resume)
- **`pg_stat_statements`** — in-DB source of truth for per-query cost.
- **`postgres_exporter`** — scrapes Postgres internals into **Prometheus**.
- **Prometheus** — time-series storage + alerting rules (e.g., "alert if p99 > 500 ms for 5 min").
- **Grafana** — dashboards and visual trends; the standard Postgres dashboards chart everything above.
- **`auto_explain`** — automatically logs the *plan* of any query over a threshold, so you catch a bad plan in prod without reproducing it:
  ```ini
  auto_explain.log_min_duration = '500ms'
  auto_explain.log_analyze = on
  ```
- **APM** (Datadog, New Relic, OpenTelemetry) — traces a slow API call *down to* the specific SQL statement, tying app latency to DB latency.
- Managed clouds: **AWS RDS Performance Insights**, GCP Query Insights — hosted equivalents of the above.

### 6.4 A healthy loop
`Monitor (Grafana alert) → Identify worst query (pg_stat_statements) → Diagnose (EXPLAIN ANALYZE) → Fix (index / rewrite / stats) → Measure (before/after, JMeter) → Deploy → Watch dashboards confirm`. Being able to narrate this loop end-to-end is the strongest possible version of your resume bullet.

---

## 7. Beyond single queries (know these exist)

When query-level tuning is exhausted, the next levers are architectural — worth a sentence each so you can gesture at them:

- **Connection pooling** (PgBouncer, HikariCP) — reuse connections; cheap latency win under load.
- **Caching** (Redis) — serve hot reads from memory, skip the DB entirely. *(You already did exactly this at VertexOne for user-profile lookups — 80% latency cut.)*
- **Read replicas** — offload read traffic from the primary.
- **Partitioning** — split a huge table by range/list (e.g., by date) so scans touch one partition. Postgres declarative partitioning.
- **Denormalization / materialized views** — trade storage and freshness for read speed.
- **Right tool for the job** — full-text search belongs in a search engine, not `LIKE '%...%'`. *(Your VertexOne Elasticsearch migration — 95% search-latency cut across 0.78M records — is precisely this move: relational search doesn't scale to free-text at low latency, so you moved it to an inverted-index engine.)*

---

## 8. Interview quick-reference (talking points)

- "First I **find** the slow query — `pg_stat_statements` ranked by *total* time, because frequency × mean is the real cost — or the slow-query log."
- "Then I **diagnose** with `EXPLAIN (ANALYZE, BUFFERS)`. I look for **Seq Scans** on selective filters, big gaps between **estimated and actual rows** (stale stats), and high **Rows Removed by Filter**."
- "**Fixes** are usually a **composite index** matching the `WHERE`/`JOIN` columns with the leading column chosen for selectivity, or **rewriting** the query to be **SARGable** and set-based — killing an **N+1** loop is often the biggest win."
- "I **measure** before/after with median execution time and the plan itself — e.g., 2000 ms → 100 ms is a **95%** reduction / **20×** — and validate under load with **JMeter**, tracking **p95/p99**, not averages."
- "I keep it healthy with **Prometheus + Grafana** dashboards and alerts on latency percentiles and cache-hit ratio."
- Know the trade-off: "indexes speed reads but slow writes and cost storage, so I add the minimum that covers the hot paths."

### One-line definitions to have ready
- **Index** — sorted side structure to find rows without scanning the table.
- **Sequential scan** — read the whole table; fine when you need most of it, bad for a few rows.
- **SARGable** — a predicate an index can be used for (no function wrapping the column).
- **Selectivity / cardinality** — how much a filter narrows results / how many distinct values a column has.
- **Composite index leftmost prefix** — a multi-column index is usable from the first column rightward, not from the middle.
- **EXPLAIN vs EXPLAIN ANALYZE** — predicted plan vs actually-executed plan with real timings.
- **p95/p99 latency** — the slow tail users actually feel; averages hide it.

---

*Both PostgreSQL and MySQL share these principles; syntax differs (`pg_stat_statements`/`auto_explain` ↔ slow query log + `pt-query-digest`; `EXPLAIN` output format), but "read the plan, touch fewer rows, do less work per row, measure the delta" is universal.*