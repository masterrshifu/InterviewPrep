

# Database Indexing — Complete Guide (Learning + Interviews)
 
---

## 1. What an Index Is (and Why)

An **index** is a separate data structure that stores a sorted (or hashed) copy of one or more columns plus a pointer back to the full row. It exists for one reason: to let the database **find rows without scanning the whole table**.

**Book analogy:** A table without an index is like a book with no index page. To find every mention of "photosynthesis" you read all 900 pages (a *full table scan*). The index at the back lists the word once with page numbers, so you jump straight there. The tradeoff is identical: the index takes extra pages (storage) and must be updated whenever the book content changes (write cost).

**Core tradeoff to remember for interviews:**
> Indexes make **reads faster** but make **writes (INSERT/UPDATE/DELETE) slower** and **consume storage**. Every index is a bet that read savings outweigh write and space costs.
 
---

## 2. How Indexes Work Under the Hood

### Table storage first
- **Heap table:** rows stored in no particular order (PostgreSQL default). Index entries point to a physical location (tuple ID / row ID).
- **Clustered/index-organized table:** the table *is* the index — rows are physically stored in primary-key order (MySQL InnoDB, SQL Server clustered).
### The data structure does the work
The reason an index is fast is that it replaces O(n) scanning with a structure that supports O(log n) lookup or O(1) hashing. The two dominant families:

- **B-Tree / B+Tree** — balanced tree, keeps keys sorted, supports range queries. The default for almost every relational database.
- **Hash** — hash table, O(1) equality lookup, but **no range or sorting support**.
  Other engines (search, analytics, NoSQL) use inverted indexes, bitmaps, LSM-trees, etc. (covered below).

---

## 3. Types of Indexes — by Data Structure

### 3.1 B-Tree / B+Tree (the default, know this cold)
- Self-balancing tree; all leaf nodes at the same depth → predictable O(log n).
- In a **B+Tree**, all actual data/pointers live in the **leaf nodes**, and leaves are linked in a doubly linked list. Internal nodes hold only keys for navigation.
- The linked leaves make **range scans** (`BETWEEN`, `>`, `<`, `ORDER BY`) very efficient — find the start, then walk the leaves.
- Supports: equality, ranges, prefix matching (`LIKE 'abc%'`), `ORDER BY`, `MIN`/`MAX`.
- **This is what you get by default** with `CREATE INDEX`.
  **Why B+Tree over plain B-Tree in databases:** keeping all data in leaves means internal nodes are smaller → higher fan-out → shorter tree → fewer disk reads. Linked leaves make sequential/range access trivial.

### 3.2 Hash Index
- Hashes the key into buckets. **O(1) average** for exact equality (`=`).
- **Cannot** do range queries, `ORDER BY`, or prefix matching — hashing destroys ordering.
- Use when you only ever do exact-match lookups (e.g., session token lookup). In practice B-Trees are so good that hash indexes are rarely worth it; Postgres supports them but they're niche.
### 3.3 Bitmap Index
- Stores a bitmap (one bit per row) for each distinct value of a column.
- Excellent for **low-cardinality** columns (few distinct values: gender, status, boolean, country) especially in **read-heavy data warehouses**.
- Multiple bitmaps combine with fast bitwise AND/OR for complex `WHERE` conditions.
- **Bad for high-write OLTP** — updating bitmaps under concurrency is expensive (locking). Common in Oracle / analytics engines.
### 3.4 Inverted Index (full-text search)
- Maps each **term → list of documents/rows containing it**. This is how search engines and full-text search work.
- Postgres **GIN** (Generalized Inverted Index) is the classic example — great for full-text search, arrays, JSONB containment.
- Used by Elasticsearch/Lucene, and Postgres `tsvector` columns.
### 3.5 GiST / R-Tree (spatial & multidimensional)
- **R-Tree / GiST** index bounding boxes → efficient for **geospatial** and range/overlap queries ("find restaurants within 5 km", "which shapes intersect").
- PostGIS uses GiST. SQL Server / MySQL have spatial indexes built on similar ideas.
### 3.6 LSM-Tree (Log-Structured Merge Tree)
- Not a classic secondary index but a **storage engine** design optimized for **write-heavy** workloads.
- Writes go to an in-memory structure (memtable), flushed sequentially to disk as sorted files (SSTables), later merged (compaction).
- Trades read amplification for very fast writes. Used by **Cassandra, RocksDB, LevelDB, HBase, ScyllaDB**; contrast with B-Tree engines (fast reads, slower random writes).
  **Interview soundbite:** *B-Trees favor reads; LSM-Trees favor writes.*

### Quick comparison

| Structure | Best for | Range queries | Weakness |
|-----------|----------|---------------|----------|
| B+Tree | General purpose, ranges, sorting | Yes | Slower writes than LSM |
| Hash | Exact equality only | No | No ranges/ordering |
| Bitmap | Low-cardinality, read-heavy analytics | Via bit ops | Poor under heavy writes |
| Inverted (GIN) | Full-text, arrays, JSONB | N/A | Larger, slower updates |
| GiST / R-Tree | Spatial, geometric | Overlap/nearest | Specialized |
| LSM-Tree | Write-heavy stores | Yes (sorted) | Read/space amplification |
 
---

## 4. Types of Indexes — by Logical Role

These describe *what* is indexed and *how it relates to the table*, independent of the underlying structure (usually B-Tree).

### 4.1 Clustered Index (know this deeply — top interview topic)
- Determines the **physical order of rows** in the table. Because rows can only be physically sorted one way, **a table can have only ONE clustered index**.
- The leaf level of a clustered index *is the actual table data*.
- In **MySQL InnoDB**, the **primary key is the clustered index** automatically. If you define no PK, InnoDB picks a unique NOT NULL key, or generates a hidden row ID.
- **Fast** for range scans on the clustered key and for PK lookups.
- **Choice of clustered key matters a lot**: a random key (like a UUID v4) causes page splits and fragmentation on insert; an auto-increment/sequential key appends cleanly.
### 4.2 Non-Clustered (Secondary) Index
- A **separate** structure whose leaf nodes contain the indexed column(s) + a pointer to the row.
- A table can have **many** non-clustered indexes.
- The pointer is either a physical row locator (heap) or, in InnoDB, **the primary key value**. This means an InnoDB secondary-index lookup does **two** traversals: secondary index → PK → clustered index (a "bookmark lookup"). This is why a fat primary key makes every secondary index bigger.
### 4.3 Primary Index / Primary Key
- Enforces uniqueness + NOT NULL; usually backs the clustered index.
### 4.4 Unique Index
- Enforces that indexed values are distinct. Speeds lookups *and* enforces a constraint.
### 4.5 Composite (Multi-Column / Compound) Index
- Index on `(A, B, C)` — multiple columns in a defined order.
- **Leftmost-prefix rule (critical):** an index on `(A, B, C)` can serve queries filtering on:
    - `A`
    - `A, B`
    - `A, B, C`
    - and `A` combined with a range on `B`
    - but **NOT** `B` alone, `C` alone, or `B, C` — because the index is sorted by A first.
- **Column order rules of thumb:**
    1. Columns used with **equality** (`=`) should come **before** columns used with **ranges** (`>`, `<`, `BETWEEN`), because after a range the remaining columns are no longer usefully sorted.
    2. Put the **most selective / most commonly filtered** column early (nuanced — see analysis section).
    3. Match the order needed by `ORDER BY` to enable an index-ordered scan.
### 4.6 Covering Index (Index-Only Scan)
- An index that contains **every column a query needs** (in `SELECT`, `WHERE`, `ORDER BY`), so the database answers the query **entirely from the index without touching the table**.
- Huge performance win — no bookmark lookup back to the heap/clustered index.
- Achieved by adding columns to the index. In SQL Server/Postgres use `INCLUDE (col)` to add non-key "payload" columns to the leaf without affecting sort order.
- **Interview line:** *"The query is covered by the index" = the index alone satisfies it.*
### 4.7 Partial / Filtered Index
- Indexes only rows meeting a condition: `CREATE INDEX ... WHERE status = 'active'`.
- Smaller, cheaper, faster when queries always target that subset (e.g., only active users, only unshipped orders).
### 4.8 Function-Based / Expression Index
- Indexes the result of an expression: `CREATE INDEX ON users (LOWER(email))`.
- Needed because a plain index on `email` **cannot** be used for `WHERE LOWER(email) = ...`. Indexing the expression makes it sargable.
### 4.9 Dense vs Sparse Index
- **Dense:** one index entry per row.
- **Sparse:** one index entry per block/page (only possible when data is physically sorted, i.e., alongside a clustered layout). Smaller, but requires an extra scan step within the block.
### 4.10 Others worth naming
- **Spatial index** (§3.5), **Full-text index** (§3.4), **Descending index** (sorted DESC for `ORDER BY col DESC`), **Hash index** (§3.2).
---

## 5. Clustered vs Non-Clustered — The Classic Interview Table

| Aspect | Clustered | Non-Clustered |
|--------|-----------|---------------|
| Count per table | One | Many |
| Physical order | Defines it | Independent |
| Leaf node holds | Actual row data | Key + row pointer/PK |
| Lookup cost | One traversal | Two (index → row) in InnoDB |
| Best for | Range scans, PK access | Selective point lookups on other columns |
| Analogy | Dictionary (words physically in order) | Textbook index (points elsewhere) |
 
---

## 6. The Real Cost of Indexes (don't skip this in interviews)

1. **Write amplification:** every INSERT/UPDATE/DELETE must update *every* affected index. Ten indexes ≈ ten extra structures to maintain per write.
2. **Storage:** indexes can rival or exceed table size.
3. **Memory / cache pressure:** indexes compete for buffer-pool RAM.
4. **Maintenance:** fragmentation, page splits, need for rebuild/`REINDEX`/`ANALYZE`.
5. **Optimizer confusion:** too many overlapping indexes can lead the planner to poorer choices and adds planning time.
   **Guideline:** index for your actual query patterns, not "just in case." Drop unused indexes.

---

## 7. How to Analyze Which Index to Use (the key skill)

### Step 1 — Start from the queries, not the table
Indexes serve **queries**, so collect the real workload: the frequent and the slow statements. For each, look at the columns in:
- `WHERE` (filtering)
- `JOIN ... ON` (join keys)
- `ORDER BY` / `GROUP BY` (sorting/grouping)
- `SELECT` list (for covering opportunities)
### Step 2 — Judge selectivity & cardinality
- **Cardinality** = number of distinct values in a column.
- **Selectivity** = distinct values / total rows (or, fraction of rows a predicate returns). High selectivity = a predicate that returns *few* rows.
- **Index high-selectivity columns.** An index on a column where each value returns ~1 row (e.g., email, user_id) is excellent. An index on a boolean returning 50% of rows is usually **ignored** — a full scan is cheaper than an index scan + many row fetches.
- Rough rule: if a predicate selects **more than ~5–10%** of the table, the optimizer often prefers a sequential scan. B-Tree indexes shine on selective predicates; **low-cardinality** columns are better served by bitmap indexes (analytics) or left unindexed.
### Step 3 — Read the query plan (EXPLAIN)
This is the single most important practical skill. Learn your engine's tool:
- **PostgreSQL:** `EXPLAIN ANALYZE <query>` — shows estimated + actual rows, timing.
- **MySQL:** `EXPLAIN` / `EXPLAIN ANALYZE`.
- **SQL Server:** actual execution plan / `SET STATISTICS IO ON`.
- **Oracle:** `EXPLAIN PLAN` / `DBMS_XPLAN`.
  **What to look for:**
- `Seq Scan` / `Full Table Scan` on a big table with a selective filter → likely a missing index.
- `Index Scan` / `Index Seek` → index is being used. Good.
- `Index Only Scan` → covering index; ideal.
- **Estimated rows vs actual rows** wildly different → stale statistics; run `ANALYZE` / update stats.
- Expensive `Sort` step → consider an index matching the `ORDER BY`.
- Nested-loop join with a scan on the inner table → index the join key.
### Step 4 — Design the index to match access patterns
- **Single-column** for a single frequent filter.
- **Composite** when queries filter on multiple columns together — order by the rules in §4.5 (equality before range; match `ORDER BY`).
- **Covering / INCLUDE** when a hot query selects a few columns and you want to avoid row lookups.
- **Partial** when queries always hit a subset.
- **Expression** when you filter on a transformed column.
### Step 5 — Make predicates "sargable"
**SARGable** = Search-ARGument-able = the predicate can use an index. Common index-killers to avoid:
- Wrapping the column in a function: `WHERE YEAR(created_at) = 2024` → rewrite as a range `WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'` (or add an expression index).
- Leading wildcard: `LIKE '%abc'` cannot use a normal B-Tree (trailing `'abc%'` can).
- Implicit type conversion / mismatched collation on the column.
- `OR` across different columns (sometimes better as `UNION` or separate indexes).
- Applying arithmetic to the column: `WHERE price * 1.1 > 100`.
### Step 6 — Weigh read vs write and consolidate
- On write-heavy tables, minimize index count.
- **Consolidate:** an index on `(A, B)` already covers queries on `A`, so you may not need a separate index on `A`. Remove redundant/overlapping indexes.
- Use tooling to find **unused indexes** (e.g., Postgres `pg_stat_user_indexes`, SQL Server missing/unused-index DMVs) and drop them.
### Step 7 — Test with realistic data volume
Optimizer choices depend on data distribution and size. A plan on 1,000 rows differs from 100M rows. Test against production-scale data, keep statistics fresh (`ANALYZE`/auto-stats), and re-check plans after changes.

### A compact mental checklist
1. Which columns appear in `WHERE` / `JOIN` / `ORDER BY`?
2. How selective is each (cardinality)?
3. Run `EXPLAIN` — is it scanning or seeking?
4. Single vs composite? What column order (equality → range → sort)?
5. Can I make it covering?
6. Is the predicate sargable?
7. What's the write cost, and is there a redundant index to drop?
---

## 8. Common Interview Questions (with crisp answers)

**Q: What is an index and its main tradeoff?**
A separate sorted/hashed structure for fast lookup. Speeds reads; costs write performance and storage.

**Q: Clustered vs non-clustered?**
Clustered defines physical row order, one per table, leaf = the data. Non-clustered is separate, many allowed, leaf = key + pointer to row. (See §5.)

**Q: Why B-Tree (B+Tree) instead of a hash for a general index?**
Hash is O(1) for equality only. B+Tree supports ranges, ordering, prefix matches, and MIN/MAX with O(log n) — far more versatile for real queries.

**Q: How does a composite index work / leftmost prefix rule?**
Index on `(A,B,C)` is sorted by A, then B, then C. It serves prefixes `A`, `A,B`, `A,B,C` — not `B` or `C` alone. Put equality columns before range columns.

**Q: What is a covering index?**
An index containing all columns a query needs, so it's answered from the index alone (index-only scan) without touching the table.

**Q: When would an index NOT help / be ignored?**
Low selectivity (predicate returns a large fraction of rows), non-sargable predicates (function on column, leading wildcard), very small tables, or when a sequential scan is cheaper. Also on write-heavy tables where the maintenance cost outweighs read gains.

**Q: Why can too many indexes hurt?**
Every write must maintain them all; they consume storage and cache and can slow/confuse the optimizer.

**Q: How do you decide which index to add?**
Look at real query patterns, judge selectivity, run `EXPLAIN`, design a matching (possibly composite/covering) index, ensure predicates are sargable, and weigh write cost. (See §7.)

**Q: What's a sargable query?**
One whose `WHERE` predicate lets the engine use an index — the indexed column is compared directly, not wrapped in a function or a leading wildcard.

**Q: Difference between primary key and unique index?**
PK = unique + NOT NULL, one per table, typically clustered. Unique index enforces uniqueness but allows (typically one) NULL and you can have several.

**Q: B-Tree vs LSM-Tree storage engines?**
B-Tree: fast reads, in-place updates, good for read-heavy OLTP. LSM-Tree: sequential writes + compaction, great for write-heavy workloads (Cassandra, RocksDB), at the cost of read/space amplification.

**Q: How does InnoDB store rows and what does a secondary index point to?**
Rows are clustered by PK. Secondary indexes store the PK value (not a physical pointer), so a secondary lookup does index → PK → clustered-index traversal. Keep the PK small.

**Q: What happens to indexes on INSERT?**
Every index on the table is updated, which is why heavy indexing slows writes and can cause page splits/fragmentation (worse with random keys like UUIDv4).
 
---

## 9. Best Practices & Pitfalls (cheat sheet)

**Do**
- Index columns used in `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY`.
- Prefer selective, high-cardinality columns.
- Use composite indexes with the right column order (equality → range → sort).
- Consider covering/`INCLUDE` for hot read paths.
- Keep primary keys small and preferably sequential.
- Keep statistics fresh; read `EXPLAIN` plans.
- Drop unused / redundant indexes.
  **Avoid**
- Indexing everything "just in case."
- Functions/expressions on indexed columns in `WHERE` (use expression indexes instead).
- Leading `%` wildcards.
- Indexing very low-cardinality columns with B-Trees (consider bitmap or skip).
- Random UUID primary keys on write-heavy tables (fragmentation).
- Redundant indexes already covered by a composite's leftmost prefix.
---

## 10. One-Paragraph Summary (for quick revision)

An index is an auxiliary structure that trades write speed and storage for faster reads. Most indexes are B+Trees (sorted, range-friendly, O(log n)); alternatives include hash (equality only), bitmap (low-cardinality analytics), inverted/GIN (full-text), GiST/R-Tree (spatial), and LSM-Trees (write-heavy engines). Logically, indexes are clustered (one per table, defines physical order) or non-clustered (many, separate), and can be composite (leftmost-prefix rule, equality-before-range ordering), covering (answers a query from the index alone), partial, unique, or expression-based. To choose an index, start from your actual queries, evaluate column selectivity, read the `EXPLAIN` plan to see whether the engine scans or seeks, design a composite/covering index that matches the access pattern, keep predicates sargable, and balance read gains against write and storage cost — dropping anything redundant or unused.