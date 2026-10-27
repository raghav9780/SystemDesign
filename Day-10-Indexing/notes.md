# Day 10 — Indexing
**Phase 1: Foundations** | Score: 9.5/10

> **One-sentence idea:** An index is a **separate, sorted structure that lets the DB *find* rows without
> scanning the whole table** — exactly like the index at the back of a textbook ("mitochondria → p.412"
> instead of reading all 800 pages). The eternal trade: indexes make **reads fast** but make **writes
> slower** and **cost storage**, because every index is a second structure the DB must keep in sync.

## 1. Why not just search the table?
- A table is physically a pile of rows. `WHERE email = 'x'` with no index → **full table scan** = read EVERY row.
  Fine on 10 rows, a disaster on 100M (seconds per query).
- An index pre-sorts by a column → the DB does a **lookup** (jump to the value) instead of a **scan** (read all).
- Payoff: O(n) → ~O(log n). On 100M rows that's ~27 steps, not 100,000,000.
- Mental model: turns "read the whole haystack" into "look up where the needle is."

---

## 2. The two great families: B-Tree vs LSM-Tree ⭐ (THE key interview probe)
These are the two ways to physically organize indexed data. They map directly onto **read-optimized vs write-optimized.**

### B-Tree — the DEFAULT (Postgres, MySQL/InnoDB, Oracle, SQL Server)
- A **balanced tree of sorted pages**. Top page: "< M go left, ≥ M go right"; each level narrows the range until
  the leaf holds the row's location. Balanced → every lookup is the same shallow depth (3–4 levels even for billions).
- **Reads:** excellent + predictable — a few page hops, O(log n).
- **Range queries:** excellent — leaves are linked in sorted order, so "Jan 1 → Jan 31" = find start, walk sideways.
- **Writes:** the catch — insert edits a page **in place**; if the page is full it must **split** (extra I/O), and
  writes land at random disk spots (random I/O = slow).
- Intuition: keeps data sorted *all the time*, paying the sort cost on every write so reads are always cheap.

### LSM-Tree — write-optimized (Cassandra, RocksDB, LevelDB, ScyllaDB; storage under many NoSQL stores)
Opposite philosophy: **never do slow random writes — just append.**
1. Write lands in an in-memory sorted **memtable** (fast, RAM).
2. When full, it's flushed to disk as one **immutable** sorted file — an **SSTable** (Sorted String Table). Written
   once, sequentially (fast), never modified.
3. Many SSTables pile up → a background **compaction** merges them, discarding overwritten/deleted values.
- **Writes:** excellent — sequential appends, the fastest thing a disk does. Loved by write-heavy systems
  (time-series, logging, IoT, feeds).
- **Reads:** the catch — a value could be in the memtable or ANY SSTable → a read may check **multiple files**.
  - Mitigated by a **Bloom filter** per SSTable: a tiny probabilistic structure that says "key is *definitely not*
    in this file" → skip files that can't have it.
- Intuition: defers the sorting work — accepts writes blindly fast, cleans up later in the background.

### The trade-off in one line (say this)
- **B-Tree = read-optimized**, in-place updates, great for range scans + read-heavy OLTP.
- **LSM-Tree = write-optimized**, append-only + compaction, great for write-heavy / high-ingest.

### Read vs write amplification (precise vocabulary) ⭐ — MY NAMING MISS
- **Write amplification** — one logical write → several physical writes. LSM: memtable, then SSTable, then rewritten
  again and again during compaction.
- **Read amplification** — one logical read → several physical reads. LSM: checking multiple SSTables for one key.
  ⚠️ On Q3 I described this perfectly but didn't *name* it "read amplification" — name it next time.
- B-Trees have their own write amplification (page splits, rewriting a whole page for a tiny change) — more predictable.
- Deep truth: **you can't optimize reads, writes, AND space at once — every engine picks where to pay.** (Same
  "pick two" shape as CAP, Day 7, and Day 9's co-location dilemma.)

---

## 3. Clustered vs Non-clustered (where the actual row lives)
- **Clustered index** — table rows are **physically stored in this index's order.** Only **ONE** per table (data can
  only be sorted on disk one way), usually the primary key. Lookup lands you *directly on the row* — no second hop.
- **Non-clustered (secondary) index** — a separate structure: indexed column + a **pointer** to the row. Lookup gives
  the pointer, then a **second read** fetches the row. (In InnoDB that pointer is the PK, so a secondary lookup is
  really two index traversals.)
- Analogy: clustered = a **dictionary** (words sorted, definition right there). Non-clustered = **back-of-textbook
  index** (gives a page number, then you flip to it).
- Engine reality (pass 2): **MySQL/InnoDB** always clusters the table on the primary key. **SQL Server** lets you choose
  the clustered index. **Postgres has no maintained clustered index** — a table is an unordered "heap", and *every* index
  (including the PK) is secondary, pointing at a physical row location. (Postgres's `CLUSTER` command re-sorts the table
  once, but new rows aren't kept in order.)

---

## 4. Composite indexes & the leftmost-prefix rule ⭐ (Q1 — I nailed this)
- A **composite (multi-column) index** sorts by several columns together, e.g. `(last_name, first_name)` — sorted by
  `last_name` first, and by `first_name` only *within* each last name (exactly a phone book).
- **Leftmost-prefix rule:** the index serves only queries that include a **prefix** of its columns, left to right.
  - Index `(a, b, c)` serves: `a` ✅ · `a,b` ✅ · `a,b,c` ✅
  - Does NOT serve: `b` alone ❌ · `c` alone ❌ (skips the leftmost column).
- Why: the phone book is sorted by last name → "find everyone named *John*" is useless, the Johns are scattered
  across every last name. You can only jump in if you know the last name first.
- ⭐ **Column order is a design decision, not an afterthought.**

---

## 5. Covering index (the free speed-up) ⭐ (Q4 — I nailed this)
- A query is **covered** when *every column it needs* is already in the index → the DB answers entirely from the
  index and **never touches the table** (skips the §3 second hop).
- Example: index `(user_id, status)`; query `SELECT status FROM orders WHERE user_id = 42` needs only those two
  columns → covered → no row fetch. Fast.
- Why a wider index can be worth it: it pays storage to eliminate the row-fetch.

---

## 6. When does an index actually help? — selectivity / cardinality ⭐ (Q2 — I nailed this)
- An index is worth it on a **high-selectivity** column (many distinct values → each lookup narrows to a tiny fraction).
  - `email` → ~unique → super selective → index is gold.
  - `gender` / `is_active` (boolean) → 2 values → low selectivity → index returns ~half the table, so the DB often
    **ignores it and scans** (a scan beats millions of pointer-chases). Same cardinality idea as choosing a shard key (Day 9).
- Rule of thumb: index columns you **filter/join/sort on** AND that are **selective**. Don't index everything.

---

## 7. Specialized indexes (know they exist + when)
- **Inverted index** — powers **full-text search** (Elasticsearch, Lucene, Postgres GIN). Stores "**word → list of
  docs containing it**" instead of "doc → words." So "find docs containing *database*" is a direct lookup. The
  back-of-book index, flipped.
- **Geospatial indexes** — a B-Tree can't index 2D location (you can sort by lat OR long, not "nearness"). Encode 2D
  into something searchable: **geohash** (interleave lat/long into one string → nearby points share prefixes),
  **quadtree** (recursively split the map into 4 quadrants), **R-tree** (bounding boxes). For "restaurants near me."

---

## 7b. Other index TYPES / variants you should know (the practical toolkit)
§2–§7 covered the *engine* (B-Tree/LSM) and *structure* (clustered, composite, covering, specialized). These are the
everyday **variants** you create with `CREATE INDEX` — interviewers expect you to know they exist:
- **Unique index** — enforces no-duplicates on a column *and* speeds lookups (every PK / `UNIQUE` constraint is
  backed by one). Two jobs in one structure: correctness + speed.
- ⭐ **Partial / filtered index** — index only the rows matching a predicate, e.g.
  `CREATE INDEX ... ON orders (created_at) WHERE status = 'pending';`. Small + cheap because it skips the irrelevant
  majority. **The fix for "I query a rare value of a low-cardinality column"** (see the RESOLVED follow-up). SQL Server
  calls it a "filtered index."
- **Hash index** — backed by a hash table, not a tree. O(1) **exact-match** lookups (`WHERE x = ?`) — but **no range
  queries and no sorting** (a hash scatters order). Use only when you never need ranges. (Postgres has them. Note:
  an LSM **memtable is a *sorted* in-memory structure** — a skip list or red-black tree, e.g. RocksDB's skiplist —
  precisely so it can be flushed to disk as a sorted SSTable in key order. Only the per-SSTable **Bloom filters** are
  hash-based.)
- **Expression / functional index** — index the *result of a function*, e.g. `CREATE INDEX ON users (LOWER(email))`
  so `WHERE LOWER(email) = ?` can use an index (a plain index on `email` couldn't, because the function changes the value).
- **Full-text index** = the **inverted index** (§7). **Geospatial index** = geohash/quadtree/R-tree (§7).

> Takeaway: "add an index" isn't one choice — you pick the *type* to match the query shape (exact vs range vs rare-value vs computed).

---

## 8. The cost — why you don't index everything
- Every index is a second structure to keep in sync → every `INSERT`/`UPDATE`/`DELETE` must update **every** index on
  the table → writes slow down, storage grows. Five indexes on a write-heavy table can cripple it.
- Discipline: an index is a deliberate trade — **faster reads bought with slower writes + more disk.** Add them for
  queries that are slow *and* frequent; don't sprinkle them everywhere.

---

## 🎯 The task & my answers (e-commerce orders, 500M rows)

**Q1 — `WHERE user_id = ? ORDER BY created_at DESC`:** composite index **`(user_id, created_at)`**, in that order.
Filter by `user_id` first, then `created_at`, per leftmost-prefix; reversing it forces a scan because a
`user_id`-only query can't use `created_at` as the leftmost column. ✅ **Bonus:** the index is already sorted by
`created_at` within each user → it also satisfies `ORDER BY created_at DESC` **for free** (no separate sort step).

**Q2 — index on `status`?** No — **low cardinality** (4 values). A plain index returns nearly half the rows as
pointers, so the DB scans instead. (See the RESOLVED follow-up below for the nuance.) ✅

**Q3 — write-heavy engine:** **LSM-Tree.** Writes are sequential appends (memtable → immutable SSTable →
background compaction) = fastest disk op. **Cost I accept = read amplification** (a key may sit in the memtable or
any SSTable → reads touch several files); mitigate with **Bloom filters**. ✅ (described it; didn't name it — fix.)

**Q4 — answer `SELECT status WHERE user_id=? AND order_id=?` without touching the table:** a **covering index**
`(user_id, order_id, status)` — every needed column lives in the index, so no row fetch. ✅

---

## ⭐ Follow-up (RESOLVED) — would an index on `status` ever be worth it?

**Yes — a PARTIAL (filtered) index.** Postgres: `CREATE INDEX ... ON orders (created_at) WHERE status = 'pending';`
- The low-cardinality objection is about *cardinality across the whole table*. But the nightly job only wants the
  **tiny sliver** that is `pending` (most orders quickly become shipped/delivered).
- A **partial index** only indexes the rows matching the predicate → it's small, cheap to maintain, and points
  straight at exactly the pending rows. It sidesteps low cardinality because it never indexes the huge
  shipped/delivered majority at all.
- Same idea, other engines: SQL Server "filtered index." General lesson: **low cardinality kills a *full* index, but
  if the value you query is rare, a partial index on just that value is excellent.**

---

## Personal Notes (recurring gaps)
- [ ] **Name the cost, not just describe it:** I explained the LSM read penalty but didn't call it **read
  amplification** — and the skip-trick is a **Bloom filter**. Naming = the 9→10 jump (signature recurring lesson).
- [x] **Composite index + leftmost-prefix = 3/3** ✅ (incl. the free `ORDER BY` sort).
- [x] **Selectivity/cardinality named correctly** ✅.
- [x] **Covering index named + mechanism** ✅.
- [x] **LSM mechanism (memtable → SSTable → compaction) correct** ✅.

---

## ⭐ Interview-completeness additions (audit pass)

**1. EXPLAIN / the query planner (the biggest gap — this whole doc talked about *making* indexes, never about the DB *choosing* them).** Creating an index only makes it **available**; a **cost-based query planner** then decides whether to actually use it, comparing the estimated cost of an **index scan** vs a **seq scan** using **table statistics** (row counts, histograms of value distribution, selectivity). `EXPLAIN` shows you the plan the planner picked; `EXPLAIN ANALYZE` runs it and shows real timings. You want to see **Index Scan** or **Index Only Scan**; a **Seq Scan** means your index was ignored or is missing. Classic gotchas for "I made an index but it's not being used": **stale stats** (run `ANALYZE`), a **function or type mismatch** on the column that disables the plain index (see the expression-index note in §7b), or **low selectivity** making a full scan genuinely cheaper (the planner is right — the index just isn't worth it here).

**2. B-Tree vs B+Tree (§2 actually describes a B+Tree but never names the distinction).** In a plain **B-Tree**, data / row-pointers live in **every** node. In a **B+Tree**, all values live only in the **leaf** level — internal nodes hold **only keys as a routing index**, and the leaves are chained together as a **sorted doubly-linked list**. Two wins: **wider fan-out** (internal nodes pack more keys → shallower tree → fewer disk reads), and **range scans walk the leaf links sideways** (exactly the "find start, walk sideways" trick in §2). When Postgres or InnoDB says "B-Tree index," it really means a **B+Tree**.

**3. LSM uses a WAL / commit log (the missing durability piece).** The memtable lives in **RAM** — so a crash before flush would lose it. Fix: every write is **first appended sequentially to an on-disk write-ahead log (WAL)**, and *then* placed in the memtable. On crash before the SSTable flush, the memtable is **rebuilt by replaying the WAL**. This is why LSM writes are durable despite living in memory — and note it's a **second sequential write**, so it's part of write amplification too.

**4. Compaction strategies — size-tiered vs leveled (§2 said "compaction" but not *how*).** **Size-tiered** (Cassandra default): merge SSTables of **similar size** together. Cheap writes, but a single key can live in **many** tables at once → higher **read + space** amplification. **Leveled** (RocksDB / LevelDB): keep SSTables in **levels of increasing size** (each level ~10× the previous), where the tables *within* a level have **non-overlapping** key ranges, so a key sits in **at most one table per level** (the small first level, L0, is the exception — freshly flushed files may overlap) → lower read + space amplification, but far more background merging → **higher write** amplification. The read-vs-write-vs-space "pick your poison" from §2, made concrete: even *within* LSM you choose which axis to pay.

**5. Space amplification (the named third axis).** = **physical bytes on disk ÷ logical size of the data**. LSM inherently carries it because superseded and deleted values **linger until compaction** removes them. Key detail: an LSM **delete is a TOMBSTONE** — a marker written *on top*, not an in-place removal — so a delete is *another write* and the old bytes stay until a merge cleans them. This completes the trio named in §2: **read, write, and space amplification — no engine minimizes all three.**

**6. Index-only scan (the plan keyword for a covering index, §5).** When a query is **covered**, the planner reports an **"Index-Only Scan."** Caveat in Postgres: it still consults the **visibility map**, so a covering index fully avoids the heap **only when the table is well-vacuumed**. Postgres and SQL Server also let you attach payload columns via **`INCLUDE(...)`** — they ride along in the leaf to *cover* the query **without widening the sort key** (keeps the searchable part of the index lean).

**7. Local vs global secondary indexes under sharding (the Day 9 ↔ Day 10 bridge).** Once a table is sharded, a secondary index can be organized two ways. **Local (document-partitioned):** each shard indexes only its **own** rows → **cheap writes** (all local), but a query on that index must **scatter-gather** across every shard (real examples per DDIA ch. 6: **MongoDB, Cassandra, Elasticsearch/Solr, Riak, VoltDB**). ⚠️ *Pass-2 correction:* this is **not** DynamoDB's LSI — an **LSI** keeps the table's **same partition key** with an alternate sort key, every LSI query must name that partition key, so it hits **one** partition and never scatters (but also can't search across partitions). **Global (term-partitioned):** the index is partitioned **by the indexed term**, so entries may live on **other** shards → **fast, targeted reads** (go straight to the one partition), but writes must update **remote** index partitions and are typically **eventually consistent** (DynamoDB **GSI** — AWS: updated asynchronously, eventually consistent reads only).

**8. Polish / naming.** For the **inverted index** (§7): the "word → docs" list is properly called a **postings list**, and the step that breaks text into terms is **tokenization / analysis**. For **geospatial** (§7): alongside geohash / quadtree / R-tree, name-drop **S2** (Google) and **H3** (Uber) — hierarchical cell systems that are the modern production answer to "index the globe." (Fully taught in pass-2 §11 below.)

---

## ⭐ Completeness additions (pass 2)

### 9. Probabilistic data structures — "approximately right, tiny, and fast"
**The idea:** sometimes an *exact* answer needs huge memory ("have I seen this URL among 10 billion?", "how many
*unique* visitors today?"). A probabilistic structure gives an answer that is **slightly fuzzy in a known, controlled
way**, using a tiny fraction of the memory. Interviews use three of them.

#### 9a. Bloom filter — "is X in the set?" → *definitely not* / *maybe yes*
**Mechanism.** A Bloom filter is just a **bit array of m bits** (all start at 0) plus **k different hash functions**.
- **Add X:** run X through all k hashes → get k positions → set those k bits to 1.
- **Check Y:** run Y through the same k hashes → look at those k bits.
  - If **any** bit is 0 → Y was **definitely never added** (adding Y would have set it).
  - If **all** are 1 → Y is **probably** in the set — but those bits might have been set by *other* items.

**Tiny worked example** (m = 10 bits, k = 3):
- Add `alice` → hashes 1, 4, 7 → bits 1,4,7 = 1. Add `bob` → hashes 2, 4, 9 → bits 2,4,9 = 1.
- Check `dave` → hashes 0, 4, 7 → bit 0 is **0** → *definitely not present*. ✅ No disk read needed.
- Check `carol` → hashes 1, 2, 9 → all **1** (set by alice and bob) → says "maybe" — but carol was never added.
  That's a **false positive**.

**The two guarantees to say out loud:**
- **No false negatives** — if it says "not present", it's right, always.
- **False positives possible**, at a rate you **tune by size**: more bits per item → fewer false positives. Rule of
  thumb from the standard formula: ~**9.6 bits per item ≈ 1%** false positives (with ~7 hash functions); ~14.4 bits ≈ 0.1%.
  So 1 billion keys at 1% ≈ 1.2 GB — versus tens of GB to store the keys themselves.

**Why you can't delete:** to remove `alice` you'd clear bits 1,4,7 — but bit 4 is shared with `bob`, so now `bob` would
wrongly look absent (a false negative — breaking the core guarantee). Fix: a **counting Bloom filter** stores a small
**counter** per slot instead of one bit; add increments, delete decrements, and a slot reads "set" while its count > 0.
Costs several times more memory.

**Where it's used:** LSM-trees keep one per SSTable (Cassandra, RocksDB, HBase) → skip files that can't contain the key
(cuts read amplification, §2). Also: "has this user already seen this post?", web crawlers ("already crawled this URL?"),
CDN "only cache on the second request" (one-hit-wonder filtering), malicious-URL pre-checks.
> Interview one-liner: *"A Bloom filter is a bit array plus k hashes; it never gives false negatives, its false-positive
> rate is set by the bits per item, and plain ones can't delete — use a counting Bloom filter for that."*

#### 9b. Count-min sketch — "how many times have I seen X?" (approximately)
**Mechanism.** A small 2-D grid of counters: **d rows × w columns**, each row with its **own hash function**.
- **Add X:** in each row, hash X to a column and **increment** that counter (d increments total).
- **Count X:** look at X's counter in each row and take the **minimum**.

**Why minimum, and why it only ever OVER-estimates:** a counter can be shared by X and other items that collided into
it, so each row's counter is X's true count **plus** some collision noise — never less. Taking the min across rows picks
the row with the *least* collision noise. So the estimate is **≥ the true count** — it can overcount, never undercount.
(Sizing, from the original paper: width w ≈ e/ε and depth d ≈ ln(1/δ) give error ≤ ε × total-items with probability 1−δ.)

**Where it's used:** **heavy hitters / Top-K** — "top 10 trending hashtags", "top searched queries in the last minute".
Keep a count-min sketch for frequencies + a small **min-heap of size K**: for each event, update the sketch, read its
estimate, and if it beats the heap's smallest, swap it in. Fixed memory no matter how many distinct hashtags exist.
Also: spotting abusive IPs for rate limiting (Day 14).

#### 9c. HyperLogLog — "how many DISTINCT items?" (cardinality)
**Intuition.** Hash every item to random-looking bits. A hash that starts with **one** leading zero turns up for half of all
items; **ten** leading zeros turns up about once per 2^10 = 1,024 distinct items. So "the longest run of leading zeros
I've seen" hints at how many distinct items passed by — and duplicates can't inflate it (the same item always hashes
the same). One such guess is noisy, so HLL splits items into many buckets ("registers"), keeps the max per bucket,
and averages them (a harmonic mean) for a stable estimate.

**In Redis:** `PFADD visitors:2026-10-27 user123` adds an item; `PFCOUNT visitors:2026-10-27` returns the estimated unique
count; `PFMERGE` combines HLLs (e.g. 7 daily ones → weekly uniques; the result estimates the union's distinct count).
Per the Redis docs: **at most 12 KB per key**, **standard error 0.81%**, whether you count a thousand or a billion uniques.
(Exact counting with a set of 100M user IDs would need gigabytes.)

**Where it's used:** unique visitors per page/day, distinct search queries, distinct IPs — any "count unique" where ±1% is fine.

**Summary (after the teaching):**

| Structure | Question it answers | Error type | Memory |
|---|---|---|---|
| Bloom filter | "Is X in the set?" | False positives only (no false negatives) | ~10 bits/item for 1% |
| Counting Bloom | Same, **with deletes** | Same | Several × a Bloom filter |
| Count-min sketch | "How often has X appeared?" | **Overestimates only** | Fixed d × w counters |
| HyperLogLog | "How many distinct items?" | ±0.81% std error (Redis) | ≤ 12 KB (Redis) |

---

### 10. 🛠️ Hands-on: reading `EXPLAIN` in Postgres (Rule F)
**The idea:** you never *guess* whether an index helps — you **ask the planner** and read its plan. `EXPLAIN` shows the
plan it *would* use (doesn't run the query); `EXPLAIN ANALYZE` **actually runs** it and adds real timings.
⚠️ Because `EXPLAIN ANALYZE` executes the statement, wrap an `UPDATE`/`DELETE` in `BEGIN; … ROLLBACK;` when testing.

**Step 1 — build a test table (5M rows, ~100K customers → ~50 orders each):**
```sql
CREATE TABLE orders (
  id          BIGSERIAL PRIMARY KEY,
  customer_id BIGINT      NOT NULL,
  status      TEXT        NOT NULL,
  total_cents INT         NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO orders (customer_id, status, total_cents, created_at)
SELECT (random() * 100000)::int,
       (ARRAY['pending','shipped','delivered','cancelled'])[1 + floor(random() * 4)::int],
       (random() * 50000)::int,
       now() - random() * interval '365 days'
FROM generate_series(1, 5000000);

ANALYZE orders;   -- refresh the planner's statistics
```
- `generate_series(1, 5000000)` produces 5M rows; `random()` fills fake values. `ANALYZE` collects the row counts and
  value histograms the planner uses to estimate costs — without fresh stats its estimates can be badly wrong.

**Step 2 — query with NO index on `customer_id`:**
```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 4217;
```
*Illustrative output (your exact numbers will differ):*
```
Gather  (cost=1000.00..63520.43 rows=50 width=45) (actual time=1.9..288.4 rows=48 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on orders  (cost=0.00..62515.43 rows=21 width=45) (actual time=4.8..279.6 rows=16 loops=3)
        Filter: (customer_id = 4217)
        Rows Removed by Filter: 1666651
Planning Time: 0.140 ms
Execution Time: 288.6 ms
```
**How to read it, field by field:**
- **Read the tree inside-out** — the most-indented node (`Parallel Seq Scan`) runs first and feeds its parent (`Gather`,
  which collects results from parallel workers).
- **`Seq Scan`** = read *every* row. The red flag. (`Parallel` = split across worker processes; `loops=3` = the leader + 2 workers.)
- **`cost=0.00..62515.43`** = *estimated* **startup cost .. total cost**, in the planner's abstract units (relative,
  roughly "sequential page reads" — **not milliseconds**). Use it to compare plans, not as a time.
- **`rows=21`** (in the cost part) = the planner's **estimated** rows; **`width=45`** = estimated average row size in bytes.
- **`actual time=4.8..279.6`** = real **milliseconds** to first row .. last row, **per loop**. **`rows=16 loops=3`** →
  16 rows per loop × 3 loops = 48 rows total.
- **`Rows Removed by Filter: 1666651`** (per loop) → it read ~5M rows to keep 48. That's the waste an index removes.
- **`Execution Time`** = the real total. ~290 ms here.

**Step 3 — add the index and re-run:**
```sql
CREATE INDEX idx_orders_customer ON orders (customer_id);
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 4217;
```
*Illustrative output:*
```
Index Scan using idx_orders_customer on orders  (cost=0.43..203.45 rows=50 width=45) (actual time=0.031..0.190 rows=48 loops=1)
  Index Cond: (customer_id = 4217)
Planning Time: 0.210 ms
Execution Time: 0.225 ms
```
- **`Index Scan using idx_orders_customer`** — the B-Tree found the 48 entries, then fetched each row from the table.
- **`Index Cond`** (instead of `Filter`) — the condition was answered *by the index*, not by checking rows one by one.
- ~290 ms → ~0.2 ms: three orders of magnitude.
- You may instead see **`Bitmap Index Scan` → `Bitmap Heap Scan`**: the planner first collects all matching row
  locations from the index, sorts them by physical position, then reads the table pages in order. It picks this
  middle-ground when "a moderate number" of rows match (too many for row-by-row index hops, too few for a full scan).

**Step 4 — make it an Index Only Scan (covering index, §5 / addition 6):**
```sql
CREATE INDEX idx_orders_cust_status ON orders (customer_id) INCLUDE (status);
VACUUM orders;    -- updates the visibility map so Postgres can trust the index alone
EXPLAIN ANALYZE SELECT status FROM orders WHERE customer_id = 4217;
```
*Illustrative output:*
```
Index Only Scan using idx_orders_cust_status on orders  (cost=0.43..5.31 rows=50 width=10) (actual time=0.020..0.041 rows=48 loops=1)
  Index Cond: (customer_id = 4217)
  Heap Fetches: 0
```
- **`Index Only Scan`** + **`Heap Fetches: 0`** = answered entirely from the index; the table ("heap") was never touched.
  If `Heap Fetches` is large, the visibility map is stale → `VACUUM`.

**Step 5 — prove the leftmost-prefix rule (§4) to yourself:**
```sql
CREATE INDEX idx_orders_cust_created ON orders (customer_id, created_at);
EXPLAIN SELECT * FROM orders WHERE created_at > now() - interval '1 day';
```
→ expect a **Seq Scan**, not an efficient use of `idx_orders_cust_created`, because `created_at` is not the leftmost
column — the index can't jump straight to the matching rows.

**Diagnosis checklist when the plan disappoints:**
1. **Estimated rows ≫ or ≪ actual rows?** → stale/insufficient statistics → `ANALYZE`.
2. **`Seq Scan` with a big `Rows Removed by Filter`?** → missing index, or the index can't be used: function on the
   column (`WHERE LOWER(email)=…` needs an expression index, §7b), type mismatch, or skipping the leftmost column.
3. **`Seq Scan` but the filter matches a large fraction of the table?** → the planner is *right*; low selectivity (§6).
4. Add **`EXPLAIN (ANALYZE, BUFFERS)`** to see how many pages came from cache (`shared hit`) vs disk (`read`).

---

### 11. Geospatial indexing — the full picture
**The problem in one line:** "find restaurants within 2 km of me" filters on **two** numbers at once (lat AND long). A
B-Tree sorts by one value; an index on `lat` finds a thin horizontal strip around the whole planet, not a small circle.
Every geospatial index works by turning **"near in 2-D"** into something an index can find quickly.

#### Geohash — 2-D squeezed into a 1-D string
- **How it's built:** split the world's longitude range in half → write `0` (left) or `1` (right). Split latitude in half
  → `0`/`1`. Keep alternating, each bit halving the box. Every 5 bits become one base-32 character. Result: a string like
  `tdr1y` where **each extra character zooms into a smaller box** inside the previous one.
- **Why it's useful:** points that share a **prefix** are in the same box → "nearby" becomes a **string-prefix query**
  that an ordinary B-Tree handles: `WHERE geohash LIKE 'tdr1y%'`.
- **Precision (approx. cell size at the equator; cells get narrower toward the poles):**
  length **5 ≈ 4.9 km × 4.9 km** · length **6 ≈ 1.2 km × 0.6 km** · length **7 ≈ 153 m × 153 m**.
  Choose the length whose cell is at least as big as your search radius.
- ⚠️ **The edge (boundary) problem:** two cafés 20 m apart but on **opposite sides of a cell border** can have
  **different prefixes** (in the worst case — across the equator or prime meridian — no common prefix at all). Search only
  your own cell and you miss the café across the street.
  **Fix:** search your cell **plus its 8 neighbours** (the 3×3 block), then compute exact distances and filter/sort.
  Geohash libraries provide a neighbours function for exactly this.

#### Quadtree — a map that splits itself where it's crowded
- **How it's built:** start with one rectangle = the whole map. If it holds more than **K** points (say 100), **split it
  into 4 quadrants**; repeat inside any quadrant still over K.
- **Why "adaptive" matters:** downtown Manhattan ends up as many tiny cells; the middle of the ocean stays one huge cell.
  Every leaf holds a similar number of points, so "give me ~20 nearby places" costs about the same everywhere. (Geohash
  cells are **fixed-size** — a length-6 cell in Manhattan may hold thousands of places, in a desert zero.)
- **Search:** walk down to the leaf containing the user; if it has too few results, also check neighbouring leaves.
- **Trade-off:** usually an **in-memory** structure, rebuilt or carefully updated; fast-moving objects (drivers updating
  every few seconds) mean constant re-insertion → for those, fixed cells (geohash/H3) in a key-value store are simpler.

#### R-tree — boxes around groups of nearby shapes
- Groups nearby objects into **bounding rectangles**, which nest into bigger rectangles (a balanced tree, like a B-Tree for
  2-D). Handles **shapes** (delivery zones, building outlines), not just points. **PostGIS** spatial indexes use an
  R-tree-style index via Postgres's GiST index type (§12).

#### S2 and H3 (one line each)
- **S2 (Google)** — projects the Earth onto the 6 faces of a cube, recursively splits each face into 4, and numbers cells
  along a **Hilbert curve** so nearby cells usually get nearby 64-bit IDs → range scans on a normal index.
- **H3 (Uber)** — covers the globe in **hexagons** at 16 resolutions; every hexagon's 6 neighbours are the same distance
  away (squares have corner neighbours that are farther), which makes "this area and the ring around it" — surge pricing,
  demand heat-maps — clean to compute.

#### Redis GEO — the practical in-memory tool
```
GEOADD drivers 77.5946 12.9716 driver:42          # LONGITUDE first, then latitude, then member
GEOSEARCH drivers FROMLONLAT 77.59 12.97 BYRADIUS 2 km ASC COUNT 10 WITHDIST
```
- `GEOADD` stores each member in a **sorted set** whose score is a **52-bit geohash** — so it's the geohash idea, managed for you.
- `GEOSEARCH` (Redis 6.2+) returns members within a radius (`BYRADIUS`) or box (`BYBOX`), nearest first with `ASC`,
  capped by `COUNT`, with distances via `WITHDIST`. It replaces the deprecated `GEORADIUS`.
- ⚠️ Classic bug: passing **lat, long** instead of **long, lat**.
- Fit: moving drivers — re-`GEOADD` each driver every few seconds, and `GEOSEARCH` around each rider.

**Decision summary:** mostly-static places (Yelp) → geohash in a DB index, or an in-memory quadtree for density-adaptive
cells · fast-moving objects (Uber drivers) → geohash/H3 cells in Redis · polygons/shapes → R-tree (PostGIS) ·
area analytics → H3 · global cell IDs on a plain index → S2. (Phase 3 "proximity service" and "Uber" designs build on this.)

---

### 12. Postgres index types — the full menu
§7–§7b listed index *ideas*. In Postgres, `CREATE INDEX … USING <type>` picks the actual structure:
- **B-tree** (default) — equality + range + sorting. Use for the vast majority of indexes.
- **Hash** — equality only (§7b).
- **GIN** (Generalized Inverted Index) — the inverted index (§7): full-text search, **JSONB** containment
  (`WHERE data @> '{"tag":"red"}'`), array membership. Fast reads, slower writes.
- **GiST** (Generalized Search Tree) — a framework for "overlaps / contains / nearest" queries: geometric shapes and
  **PostGIS** (R-tree-like), range types, nearest-neighbour ordering.
- **SP-GiST** (space-partitioned GiST) — for structures that divide space into non-overlapping regions: **quadtrees**,
  k-d trees, tries (e.g. IP addresses, phone-number prefixes).
- **BRIN** (Block Range INdex) — stores only the **min/max per block range** of the table. Tiny, but only useful when the
  column's values **follow the physical order** of rows — e.g. `created_at` on a huge append-only events table. A query
  skips every block range whose min/max can't match.
> One-liner: *"B-tree by default; GIN for full-text/JSONB/arrays; GiST for geo and ranges; BRIN for huge time-ordered tables."*
