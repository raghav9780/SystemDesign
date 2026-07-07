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

**4. Compaction strategies — size-tiered vs leveled (§2 said "compaction" but not *how*).** **Size-tiered** (Cassandra default): merge SSTables of **similar size** together. Cheap writes, but a single key can live in **many** tables at once → higher **read + space** amplification. **Leveled** (RocksDB / LevelDB): keep **non-overlapping** SSTables organized into size-tiered **levels**, so a key sits in **at most one table per level** → lower read + space amplification, but far more background merging → **higher write** amplification. The read-vs-write-vs-space "pick your poison" from §2, made concrete: even *within* LSM you choose which axis to pay.

**5. Space amplification (the named third axis).** = **physical bytes on disk ÷ logical size of the data**. LSM inherently carries it because superseded and deleted values **linger until compaction** removes them. Key detail: an LSM **delete is a TOMBSTONE** — a marker written *on top*, not an in-place removal — so a delete is *another write* and the old bytes stay until a merge cleans them. This completes the trio named in §2: **read, write, and space amplification — no engine minimizes all three.**

**6. Index-only scan (the plan keyword for a covering index, §5).** When a query is **covered**, the planner reports an **"Index-Only Scan."** Caveat in Postgres: it still consults the **visibility map**, so a covering index fully avoids the heap **only when the table is well-vacuumed**. Postgres and SQL Server also let you attach payload columns via **`INCLUDE(...)`** — they ride along in the leaf to *cover* the query **without widening the sort key** (keeps the searchable part of the index lean).

**7. Local vs global secondary indexes under sharding (the Day 9 ↔ Day 10 bridge).** Once a table is sharded, a secondary index can be organized two ways. **Local (document-partitioned):** each shard indexes only its **own** rows → **cheap writes** (all local), but a query on that index must **scatter-gather** across every shard (DynamoDB **LSI**). **Global (term-partitioned):** the index is partitioned **by the indexed term**, so entries may live on **other** shards → **fast, targeted reads** (go straight to the one partition), but writes must update **remote** index partitions and are typically **eventually consistent** (DynamoDB **GSI**).

**8. Polish / naming.** For the **inverted index** (§7): the "word → docs" list is properly called a **postings list**, and the step that breaks text into terms is **tokenization / analysis**. For **geospatial** (§7): alongside geohash / quadtree / R-tree, name-drop **S2** (Google) and **H3** (Uber) — hierarchical cell systems that are the modern production answer to "index the globe."
