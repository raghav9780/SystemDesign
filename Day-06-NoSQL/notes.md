# Day 6 — NoSQL Deep Dive
**Phase 1: Foundations** | Score: 8/10

---

## Why NoSQL exists
- SQL pain points at scale: rigid schema, hard to scale writes horizontally, expensive joins.
- NoSQL = built for scale, flexibility, high write throughput — by RELAXING some guarantees.
- Core trade: give up some consistency/relationships → gain scalability + flexibility.

---

## The 4 NoSQL Types ⭐ (#1 NoSQL interview question — know all 4)

### 1. Key-Value 🔑
- key → opaque value (DB doesn't look inside). Simple, fast, easy to scale.
- Examples: Redis, DynamoDB, Riak.
- Use: caching, sessions/tokens, preferences, carts, counters, leaderboards.
- ❌ Can't query by anything but the key.

### 2. Document Store 📄
- Collections of JSON/BSON documents; each can differ (flexible schema). Query by fields inside.
- Examples: MongoDB, Couchbase, Firestore.
- Use: content/posts, product catalogs, user profiles — "object-shaped", evolving data.
- ❌ Weak at multi-document transactions / relationships.

### 3. Wide-Column / Column-Family 🧱
- Stored by column families; huge sparse distributed table; massive write throughput.
- Examples: Cassandra, HBase, Bigtable, ScyllaDB.
- Use: time-series, IoT, event logs, analytics, messaging history — WRITE-HEAVY at scale.
- ❌ Must design tables around query patterns IN ADVANCE; weak ad-hoc queries.

### 4. Graph DB 🕸️
- Nodes (entities) + edges (relationships); fast relationship traversal.
- Examples: Neo4j, Amazon Neptune, ArangoDB.
- Use: social graphs (friends-of-friends), recommendations, fraud detection, knowledge graphs.
- ❌ Overkill for plain bulk storage.

⚠️ LESSON: NAME THE TYPE explicitly ("document store", "wide-column", "graph"), not just the example.

---

## SQL vs NoSQL (asked nearly every interview)
| | SQL | NoSQL |
|---|---|---|
| Schema | fixed/rigid | flexible/dynamic |
| Scaling | vertical; sharding hard | horizontal, built-in |
| Consistency | strong (ACID) | often eventual (BASE) |
| Joins/relationships | excellent | weak (denormalize instead) |
| Best for | money, orders, integrity | scale, flexibility, write volume |

- Rule of thumb: **default to SQL** unless you have a specific reason (massive scale,
  flexible schema, extreme writes, or data shape fits a NoSQL type perfectly).

### Crisp interview answer (3–4 sentences — DON'T ramble):
1. Default SQL for strong consistency, transactions, relationships (money/orders/accounts, ACID).
2. NoSQL for horizontal scale, flexible/evolving schema, or extreme write throughput (BASE).
3. SQL normalizes + joins at read; NoSQL denormalizes + models data around query patterns.
4. Large systems use BOTH — polyglot persistence — right tool per workload.

---

## ACID vs BASE
- **BASE** = Basically Available, Soft state, Eventual consistency.
- Eventual consistency: no new writes → all replicas EVENTUALLY converge (not instant).
- Leads into CAP theorem (Day 7).

---

## Denormalization (the NoSQL mindset)
- SQL normalizes (no duplicate data, join at read). NoSQL DENORMALIZES (duplicate, pre-joined,
  single fast read, no joins).
- Trade: faster reads/easier scaling, BUT duplicated data → updates touch many copies + some inconsistency.
- Model data around HOW YOU'LL READ IT, not around "clean" structure.

---

## Polyglot Persistence (senior insight)
- Don't pick one DB — use the right one per job. E-commerce example:
  PostgreSQL (orders/ACID) + MongoDB (catalog) + Redis (cart/sessions) +
  Cassandra (logs) + Neo4j (recommendations).

---

## ⭐ Consistency acceptability rule (KEY correction from Q7)
- Acceptable to be eventually consistent when temporary disagreement causes NO HARM
  (e.g. follower count off by a few for 10s = vanity metric, no consequence).
- UNACCEPTABLE when correctness/money is on the line (bank balance).
- Judge acceptability by CONSEQUENCE — NOT by data size or which DB is used.

---

## ⭐ Denormalization Update Problem (follow-up — RESOLVED)
- Problem: author name/photo embedded (copied) into every post doc → user updates name →
  source collection updates but embedded copies go STALE.
- Fix = **fan-out update**: find + update every doc holding the old value.
- Do it ASYNCHRONOUSLY: update source of truth immediately (instant success to user),
  push the mass copy-update to a background job / queue (Kafka, Day 11). → eventually consistent.
  Acceptable because a briefly-stale display name causes no harm (consequence rule).

### Embed vs Reference (the real design choice)
| Approach | How | Trade-off |
|---|---|---|
| **Embed (denormalize)** | copy name/photo into each post | fast reads, but expensive fan-out + staleness |
| **Reference (normalize)** | store only author_id, look up at read | trivial updates, always fresh, but lookup per read |
- Rule: EMBED when read-heavy + rarely changes (display names) ; REFERENCE when changes often / large.
- **Hybrid (real Instagram-scale):** store author_id + cached name/photo; refresh cached
  copy via async fan-out on change. Fast reads + clean source of truth.
- PRINCIPLE: denormalization trades cheap reads for expensive writes; you own keeping copies
  in sync (async fan-out, eventual consistency). Reappears Day 34 (Twitter feed fan-out).

---

## Personal Notes (recurring gaps)
- [ ] Name the TYPE category, not just the example.
- [ ] Structure interview answers (Q6 was one run-on sentence — break into crisp points).
- [ ] Justify consistency acceptability by CONSEQUENCE, not incidental facts ("large data").
- [x] Feature→DB-type matching was 5/5 — strong polyglot instinct ✅.
