# Important Questions — Daily Revision Log


**Q:** What is the difference between an L4 and an L7 load balancer? Give one concrete routing decision that L7 can make but L4 cannot — and explain why L4 can't.

**A:** An **L4 load balancer** operates at the transport layer (TCP/UDP). It only sees connection-level info — source/destination **IP address and port** — and forwards the packet stream without inspecting the payload. It's fast and protocol-agnostic but content-blind.

An **L7 load balancer** operates at the application layer. It terminates the connection and parses the actual request (e.g. the HTTP method, URL path, headers, cookies), so it can make **content-based routing** decisions.

Concrete example only L7 can do: route `/api/*` requests to the API server pool and `/images/*` to a static-asset pool, or route based on a cookie/`Host` header. **Why L4 can't:** it never decrypts or parses the payload — the HTTP path/headers live *inside* the TCP byte stream that L4 just forwards, so that information is simply invisible to it.

---

**Q:** CAP says that during a network partition you must choose between Consistency and Availability. PACELC extends CAP with a second clause that applies even when there is no partition (the "Else / Latency / Consistency" part). What trade-off does that clause describe, and give an example of a system tuned each way?

**A:** PACELC = "if **P**artition → choose **A**vailability or **C**onsistency; **E**lse (normal operation) → choose **L**atency or **C**onsistency."

The "Else" clause says: even with a perfectly healthy network (no partition), there is *still* a trade-off — to keep replicas **strongly consistent** you must wait for them to coordinate/agree on every read or write, which **adds latency**. So you can sacrifice some consistency to get lower latency, or pay latency to stay consistent.

Examples: **DynamoDB / Cassandra are PA/EL** — they favor availability during a partition and **low latency** during normal operation (eventual consistency). **Google Spanner is PC/EC** — it favors **consistency** in both states, accepting higher latency (it waits on quorum + TrueTime). A single-leader SQL DB with synchronous replication is also EC.

---

**Q:** Explain the cache-aside (lazy-loading) pattern: what the application does on a read hit and a read miss, and what you must do on a write/update to avoid serving stale data (name the technique).

**A:** **Cache-aside (lazy loading)** — the application sits "beside" the cache and manages it directly:
- **Read hit:** app checks the cache → value present → return it. (Fast path, no DB touch.)
- **Read miss:** cache empty → app reads the **database**, **writes the value into the cache** (typically with a **TTL** so it eventually expires), then returns it. The cache fills *lazily* — only requested keys get cached.
- **Write/update:** the technique is **cache invalidation** — write to the DB, then **delete (invalidate) the cache key** for that item. The next read will miss and repopulate from the fresh DB value.

Why *delete* rather than *update* the cached entry: deleting is simpler and avoids races where two concurrent writes leave a stale value cached. A short **TTL** is the backstop that bounds staleness even if an invalidation is missed.

---

**Q:** Why does consistent hashing only require moving ~k/n keys when a node is added/removed, whereas `hash(key) % N` reshuffles almost every key when N changes? Explain the mechanism.

**A:** With **`hash(key) % N`**, the divisor **N (the node count) is part of every key's calculation.** Change N from 4 to 5 and almost every key produces a different remainder → nearly the entire dataset must relocate. Each key's home depends on *how many nodes exist*.

With **consistent hashing**, both nodes and keys are hashed onto a **ring**; a key belongs to the first node clockwise from it. A key's position on the ring is **fixed** — it depends only on `hash(key)`, never on the node count. Adding a node just drops a new point on the ring and takes over **only the arc between it and the previous node clockwise** → only the ~k/n keys in that one slice move; every other key stays put. (**Virtual nodes** — many points per physical node — keep the load even and spread a removed node's keys across many neighbors instead of dumping them all on one.)

Essence: `%N` ties every key's home to the *node count*; consistent hashing ties it to a *fixed ring position*, so node changes stay local.

---

**Q:** An order service must (a) write an order row to its SQL database and (b) publish an `OrderPlaced` event to Kafka. The naive code does `db.save(order)` then `kafka.publish(event)`. What is the name of the problem with this, what concretely goes wrong, and what standard pattern fixes it — describe how it works.

**A:** This is the **dual-write problem**: the code writes to **two separate systems** (the DB and Kafka) that share no transaction, so they can't be made to succeed-or-fail atomically. Concrete failure: `db.save` commits, then the server crashes before `kafka.publish` runs → the order exists but the event is **lost forever**, so downstream services (email, inventory, analytics) never react. (The reverse also breaks: publish succeeds but the DB commit rolls back → downstream acts on an order that doesn't exist.)

The fix is the **Outbox pattern**:
1. In the **same atomic DB transaction** as the order row, insert a row into an **outbox table** describing the event. Because it's one transaction, the order and the outbox entry commit together or not at all — atomicity is restored.
2. A separate **message relay** process (or **CDC** — Change Data Capture, e.g. Debezium tailing the DB's write-ahead log) reads unpublished outbox rows, publishes them to Kafka, and marks them done.
3. If the relay crashes mid-publish it retries on restart → **at-least-once** delivery, so downstream **consumers must be idempotent** (dedup on event ID).

Essence: convert an unsafe dual-write into a **single atomic DB write** (order + outbox row together), then let a relay/CDC reliably forward the event afterward.

---

**Q:** A B-Tree index and an LSM-Tree index handle writes differently. Explain the core structural difference in how each handles a write, use it to explain why LSM-Trees are write-optimized while B-Trees are read-optimized, and name one real database that uses each.

**A:** **B-Tree — writes update in place.** The tree is kept balanced and sorted on disk. A write must **find the correct leaf page and modify it where it lives**, often a **random disk I/O** (seek to that page, read-modify-write it), plus occasional page splits to keep the tree balanced. Because the data is always sorted and in one place, **reads are fast and predictable** — a point lookup or range scan is a short, direct traversal (O(log n)) to exactly one location. So B-Trees are **read-optimized**: you pay ordering cost on every write to make reads cheap.

**LSM-Tree — writes append, never seek.** Writes go first to an in-memory **memtable** (and a sequential write-ahead log for durability). When the memtable fills, it's flushed to disk as an immutable, sorted **SSTable** — a **pure sequential write**, no random seeks. Sequential writes are far faster than random ones, so **writes are cheap → write-optimized.** The cost is paid on **reads**: a key may live in the memtable or any of several SSTables, so a read may check multiple files (mitigated by **Bloom filters** to skip files that can't contain the key, and by **compaction** that merges SSTables in the background). Compaction also causes **write amplification** (data is rewritten repeatedly as it merges).

**The essence:** B-Tree does the ordering work **at write time** (random in-place updates) so reads are cheap; LSM-Tree defers the work, taking cheap **sequential appends** now and paying at read/compaction time. **Databases:** B-Tree → **PostgreSQL / MySQL (InnoDB)**; LSM-Tree → **Cassandra / RocksDB / LevelDB** (also HBase, ScyllaDB).
