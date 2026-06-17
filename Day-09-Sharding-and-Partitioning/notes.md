# Day 9 — Sharding & Partitioning
**Phase 1: Foundations** | Score: 7/10

> **One-sentence idea:** Sharding = **split your data into pieces, each piece on a different machine**,
> because the dataset is too big or too write-heavy for one server. Picture one giant phone book becoming
> a row of separate volumes — A–F on one shelf, G–M on the next. Each volume (shard) is a full,
> independent database holding *part* of the data.
>
> (Day 8 contrast — **replication = COPIES of the same data** for safety/reads. **Sharding = SPLITTING
> different data** for size/write capacity. Real systems do BOTH: split into shards, replicate each shard.)

## 1. Why shard
- Replication copies the WHOLE dataset onto each node — useless once the dataset won't fit on one machine,
  or writes exceed what one leader can take (all writes funnel through one leader, Day 8; read replicas don't help writes).
- Sharding fixes both: **storage** (10 TB → 1 TB × 10 shards) and **write throughput** (10 leaders, each ~1/10th the writes, in parallel).
- Cost: data is now scattered → *finding* and *combining* it across shards is the new hard problem.

## 2. Two directions to split (vocabulary)
- **Vertical partitioning** — split by **columns** (hot `user_id,name,email` in one store; huge cold `bio,avatar_blob` in another).
- **Horizontal partitioning = sharding** — split by **rows** (users 1–1M on shard A, 1M–2M on shard B; same columns, different rows). ⭐ This is "sharding" and today's focus.

## 3. Which row → which shard? (pick a SHARD KEY, then a strategy)
The **shard key** (partition key) is the column whose value decides the home shard (e.g. `user_id`). Three mapping strategies:
- **Range-based** — contiguous ranges (A–F → shard 1…). 👍 range queries easy. 👎 **uneven load** (names cluster on S/M/A); time/date ranges are worst (today's shard takes 100% of writes → hotspot).
- **Hash-based** — `shard = hash(key) % N`. 👍 **even distribution** (good hash scatters keys). 👎 loses range queries + the `%N` resharding trap (Section 4).
- **Directory-based** — a lookup table ("key X → shard 3"). 👍 max flexibility (move any key). 👎 the directory is a **SPOF** + an extra hop on every query.

## 4. The `% N` trap → consistent hashing ⭐ (Q2 — I nailed this)
- **Trap:** with `hash(key) % N`, changing N (4 shards → 5) makes **almost every key** compute a new shard →
  **mass migration** of nearly all data at once → hours-long, traffic-killing.
- **Consistent hashing (mental model):**
  - Imagine a **ring / clock face** numbered 0 → huge, wrapping around.
  - **Hash each shard** onto a point; **hash each key** onto a point.
  - A key belongs to the **first shard clockwise** from it.
  - ⭐ Add a shard → it drops on one spot and only steals keys *between it and the previous shard* →
    only ~**`k/n` keys move** (a slice), not all. Remove a shard → only its keys move to the next clockwise. Rest stays put.
- **Virtual nodes (vnodes):** give each physical shard MANY points on the ring (e.g. 200 scattered).
  - Why: one-point-per-shard gives uneven arcs + dumps a dead shard's whole load onto ONE neighbor.
  - Many points → smooth load + a dead shard's work spreads across MANY neighbors. How Cassandra & DynamoDB partition.

## 5. Choosing a GOOD shard key (makes or breaks the system)
Three properties:
1. **High cardinality** — many possible values (sharding on boolean `is_premium` = 2 buckets, useless).
2. **Even distribution** — no natural favorites overloading one shard.
3. **Query alignment** — your most common query can hit ONE shard via this key.
- ⭐ **Bad keys:** monotonically increasing (auto-increment ID / timestamp) → every new write lands on the
  *newest* shard → **write hotspot**; low-cardinality (country/status) → a few giant buckets.

### ⚠️ MY Q1 MISTAKE: range-by-username is NOT evenly distributed.
- I picked range-by-username and claimed "even distribution" — wrong: names cluster hard (S/M/A/J everywhere, almost no X/Q/Z) → lopsided, hotspot-prone.
- ✅ Right answer: **hash-based on `user_id`.** Even (hash scatters) + query-aligned (a user's tweets → one shard) + high cardinality. You only give up cross-user range scans, which Twitter never needs.
- Range partitioning earns its keep ONLY when you actually query ranges (e.g. time-series "last 7 days").

## 6. Hotspot / Celebrity problem ⭐ (the star concept) — Q3 naming miss
Two flavors of "one shard hammered while others idle" — the distinction is everything:
- **Data skew** — one shard *holds* more data → vnodes / good hashing FIX this.
- ⭐ **Access skew (the CELEBRITY problem)** — one *single key* gets enormous *traffic* (a celeb tweet, 100M followers).
  That key lives on ONE shard → that shard is crushed **even though the data is tiny.**
- ⭐ **THE TRAP:** consistent hashing + vnodes do **NOT** fix a single hot key — they balance *many* keys, but one
  white-hot key still resolves to one shard. (I correctly explained the "why" but called it just "hotspot" — name it **access skew / celebrity**.)
- **Fixes:** **cache** the hot key (Redis/CDN, Day 3) · **key-splitting/salting** (store as `celeb#1..#N` across shards, read a random one) · **dedicated shard** · **read replicas** for that hot shard.

## 7. The price of splitting: cross-shard operations
- **Cross-shard query → scatter-gather:** a query no single shard can answer (e.g. "global top-10 tweets") is sent to
  **every** shard → each returns its local result → a coordinator **merges**. ⭐ **Latency = the SLOWEST shard**
  (you wait for the laggard), and it worsens as you add shards. (Q5: I named scatter-gather + the fix, but missed "latency = slowest shard".)
  - Avoid by aligning the shard key with the query.
- **Cross-shard transactions:** Day 5 ACID was easy on ONE machine. Updating shard A *and* B atomically loses that. Options (Phase 2 deep dive):
  - **2PC (two-phase commit)** — coordinator asks all "ready?" then "commit". Correct but *blocking* + slow.
  - **Saga** — chain of local txns with **compensating** undo steps on failure.
  - Best: **design to avoid it** — co-locate related data on one shard so the txn stays local.

## 8. Distributed unique IDs — Snowflake ⭐ (Q4 — correct)
- Problem: one DB gives free unique IDs via **auto-increment** (1,2,3…). Many shards → each mints its own "1,2,3" → **collisions**.
  A central counter would be a bottleneck + SPOF.
- **Snowflake ID** (Twitter), a 64-bit number = `[ timestamp | machine/shard ID | per-machine sequence ]`:
  - **timestamp** → roughly **time-sortable** (newer = bigger) → keeps DB index inserts efficient.
  - **machine ID** → two shards never collide.
  - **sequence** → one machine mints many IDs within the same millisecond.
  - Caveat: relies on clocks moving forward → handle **clock skew / backward jumps**. (UUID/ULID are alternatives; random UUIDs hurt index locality.)

## 9. When NOT to shard (the ladder — say this in interviews)
Sharding adds permanent complexity (cross-shard queries, resharding, hot keys, distributed IDs) → it's a **last resort**:
1. **Vertical scaling** — bigger box first (Day 1).
2. **Read replicas + caching** — absorb read load without splitting (Days 3, 8).
3. **THEN shard**, once one leader genuinely can't hold the data or absorb the writes.
Sharding prematurely = classic over-engineering.

## 10. Real-system map
- **Cassandra / DynamoDB** → consistent hashing + virtual nodes (automatic).
- **MongoDB** → range or hashed shard key + a balancer that moves chunks.
- **Vitess (YouTube/MySQL), Citus (Postgres)** → sharding layers on top of SQL.

---

## ⭐ Follow-ups (RESOLVED)

### F-1. Live resharding 8 → 16 shards with ZERO downtime
- **Principle:** never route a key to a shard until it's fully caught up — **copy first, flip last.** (How MongoDB balancer / Vitess work.)
1. **Pick a low-movement split** — consistent hashing means each new shard claims a *slice* of one existing arc → only those keys move (~half of each affected shard), not everything.
2. **Backfill** — bulk-copy the affected key-ranges old→new **in the background**; old shard keeps serving ALL traffic (users notice nothing).
3. **Catch up** — new writes still hit the old shard during backfill, so also **stream the change-log (replication log / CDC, Day 8)** old→new so the new shard tails in-flight writes to ~0 lag.
4. **Verify** — row counts / checksums match, lag ≈ 0.
5. **Cutover** — atomically flip the routing (ring/directory) **one slice at a time** (tiny blast radius). During the ms flip: briefly freeze writes on just that slice, OR use a **fencing/version token** (Day 8) so a stray write to the old shard is rejected → no double-write/split-brain.
6. **Cleanup** — delete moved data from the old shard once the slice is confirmed healthy.
- One-liner: copy-then-cutover, incremental per range, old shard authoritative until the flip.

### F-2. A DM spans two users on DIFFERENT shards (the co-location dilemma)
- No free lunch — you trade one cost for another:
  - **Store by sender (A's shard)** → B reading their inbox must scatter-gather across every sender's shard. (Symmetric if by recipient.) ❌ bad for the reader.
  - ⭐ **Dual-write / denormalize (copy on BOTH shards)** — Day 6 denormalization. Each user reads from their OWN single shard → fast reads (reads dominate messaging). Cost: **2× storage + fan-out write → eventual consistency**. ✅ what most chat systems do.
  - **Shard by `conversation_id`** (deterministic hash of the sorted pair) → whole thread on one shard, single-shard reads, no cross-shard writes. Cost: a user's *list* of conversations is now scattered → need a **separate per-user conversation index**.
- ⭐ **The tension:** you can't have all three of — (a) each inbox served from one shard, (b) a single non-duplicated copy, (c) no cross-shard work. **Pick two.** Messaging usually sacrifices (b): dual-write + eat storage/eventual consistency, because inbox read latency is what users feel.

---

## Personal Notes (recurring gaps)
- [ ] **Don't claim a property you didn't check:** range-by-name is NOT evenly distributed — the brief warned exactly this. Test the key against all 3 properties honestly.
- [ ] **Naming precision:** it's **access skew / celebrity problem**, not just "hotspot"; the scheme is called **Snowflake**.
- [ ] **Answer every sub-part:** Q5 asked what *sets* the latency → **the slowest shard**; I didn't pin it.
- [x] **Consistent hashing + vnodes = strong** ✅ (disaster + ring fix + why only k/n keys move). Distributed-ID formula correct ✅. Hot-key "why" + fixes correct ✅.
