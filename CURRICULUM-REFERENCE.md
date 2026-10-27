# System Design 60-Day Program — Master Reference (Phase 1 Audit + Curriculum Build)

Source of truth for mentor "Alex". Consolidates research across 18 clusters. Use this to write complete daily briefs, set tasks, and grade. Audience: absolute beginner; teach mechanism + interview framing, not trivia.

---

## 1. Executive Summary — Top 10 Highest-Priority Gaps (ranked)

1. **No answer framework day.** Days 1-15 teach components but never the spine that drives a 45-min interview: functional vs non-functional requirements → core entities → API → high-level design → deep dives → bottlenecks. Add as **Day 16 (opens Phase 2)**. Highest leverage gap.
2. **Consensus, leader election & coordination entirely absent.** No Raft, quorum/majority, split-brain prevention, "just use etcd/ZooKeeper/Consul." Senior-level table stakes. Add a **dedicated Distributed Primitives day**.
3. **Distributed transactions (2PC vs Saga) + Outbox/CDC missing everywhere.** Near-guaranteed microservices question; pairs with sharding (cross-shard writes) and queues (dual-write). Add a dedicated day in late Phase 1 / early Phase 2.
4. **No Security & Auth day.** authn vs authz, OAuth2/OIDC, JWT (+ revocation), RBAC/ABAC, TLS 1.3, mTLS, password hashing (bcrypt/Argon2), secrets mgmt, zero-trust. Glaring for 2025-26. Add dedicated day.
5. **No Observability day.** 3 pillars (logs/metrics/traces), Golden Signals + RED/USE, SLI/SLO/SLA + error budgets, Prometheus/Grafana/Jaeger/OpenTelemetry, correlation IDs. Add dedicated day.
6. **No Resilience patterns day.** Circuit breaker (closed/open/half-open), bulkhead, retries + exponential backoff + **jitter**, timeouts/deadline propagation, graceful degradation, cascading-failure prevention. Add dedicated day.
7. **Availability/reliability math missing.** The "nines" → downtime/year table, SLA/SLO/SLI, availability in series (multiply) vs parallel/redundancy (1−(1−a)(1−b)). Add to **Day 1** and reinforce on **Day 15**.
8. **Delivery semantics + idempotency under-taught.** at-most/at-least/exactly-once ("exactly-once is a myth = at-least-once + idempotent consumer"), idempotency keys backed by DB unique constraint. Spans Day 5, 11, 12 — must be explicit and cross-linked.
9. **Consistent hashing never named as a first-class topic.** Implied in Day 9 only; interviewers ask for it by name (sharding, caching, LB). Make explicit on Day 9, cross-ref Day 2 & 3.
10. **Geospatial indexing + WebSockets/real-time + Snowflake IDs + vector DBs missing.** Each blocks multiple Phase 3 designs (Uber/Yelp, chat/notifications/Drive, sharded writes/URL shortener, semantic search/RAG). Seed in Phase 1 (Days 6, 10, 12, +9).

Honorable mentions: lost update / write skew (Day 5); leaderless replication + quorum W+R>N (Day 8); cache penetration/avalanche + hot-key (Day 3); fan-out push/pull/hybrid (Phase 2 before Twitter); deployment strategies blue-green/canary/rolling (new day).

---

## 2. Per-Cluster Reference

### Cluster 1 — Scalability (Day 1)
**Must-teach checklist**
- Performance vs scalability; latency vs throughput; tail latency p50/p95/**p99** (not averages).
- Vertical (scale up): simple, hard ceiling, expensive, still SPOF, resize downtime. Horizontal (scale out): near-linear, fault-tolerant, needs LB + partitioning + distributed complexity.
- Stateless vs stateful; **how** to externalize state (Redis/DB/object store, cookie/JWT); sticky vs token sessions.
- SPOF identification → eliminate via redundancy/replication.
- **Availability "nines"** → downtime/yr (99%=3.65d, 99.9%≈8.76h, 99.99%≈52.6min, 99.999%≈5.26min); SLA vs SLO vs SLI.
- **Availability math:** series multiply (A1×A2); parallel/redundant 1−(1−A1)(1−A2).
- Redundancy/failover: active-passive vs active-active, N+1, automated vs manual failover.
- Object/blob storage: object vs block vs file; flat namespace + metadata; storage tiers (hot/warm/cold/archive); store blob externally + URL/pointer in DB; **presigned URLs** for direct client up/download; pair with CDN.
- Read-heavy vs write-heavy + read/write ratio as a scaling lens.
- Elasticity/autoscaling metrics (CPU, RPS, queue depth); stateless tier easy, stateful (DB) tier is the hard part (→ replicas → shard).
- Optional: AKF Scale Cube (x clone / y functional / z data partition).

**Common questions:** vertical vs horizontal & when; what is stateless & why it matters; scale 1→millions evolution; find/remove the SPOF; latency vs throughput; 99.99% downtime/yr; combined availability of dependent 99.9% services; sessions break behind 2nd server — fix; store/serve images at scale; read- or write-heavy & impact; what is autoscaling/metrics; limits of vertical scaling.

**Gaps vs plan:** ADD nines + availability math; ADD latency/throughput + perf/scalability explicit defs + tail latency; EXPAND object storage (block/file distinction, tiers, URL-in-DB, presigned URLs, CDN); ADD state-externalization mechanics; ADD redundancy/failover patterns; ADD read/write-ratio lens; ADD autoscaling/elasticity; OPTIONAL AKF cube.

---

### Cluster 2 — Load Balancers (Day 2)
**Must-teach checklist**
- L4 vs L7 (route on IP/port vs HTTP headers/path/cookie/host); L4 faster/protocol-agnostic, L7 content-aware; common two-tier (L4 → L7).
- Algorithms: round robin, weighted RR, least connections, weighted least conn, least response time, IP hash, random, **power-of-two-choices (P2C)**, **consistent hashing** — and WHEN each.
- Sticky sessions (LB cookie vs app cookie vs IP-hash); downsides; preferred fix = externalize state (Redis) → stateless.
- Health checks: active vs passive, HTTP/TCP probes, healthy/unhealthy thresholds, interval/timeout, pool add/remove.
- **LB is a SPOF → make it HA:** VIP/floating IP, active-passive vs active-active, VRRP + keepalived, gratuitous ARP failover ("who balances the balancer").
- Consistent hashing in LB context: hash ring + virtual nodes, minimal remap, cache/server affinity.
- **GSLB:** DNS-based vs Anycast-based global routing, geo/latency routing, multi-region failover (taught at Day 2/4 boundary).
- **SSL/TLS termination** vs passthrough/re-encryption; only L7 terminates TLS & inspects HTTP.
- **Connection draining / deregistration delay** for zero-downtime deploys/scale-in.
- HW vs SW (F5 vs HAProxy/Nginx/Envoy); cloud: **AWS ALB=L7, NLB=L4**, ELB.
- Large uploads/WebSocket/long-lived connection handling.

**Common questions:** L4 vs L7 & when; algorithms & RR vs least-conn vs consistent hashing; sticky sessions problems & how to avoid; active vs passive health checks; make the LB HA; consistent hashing + vnodes; GSLB nearest/healthy DC; where TLS terminates; connection draining; DNS-based LB limits; large uploads/WebSockets through LB; full request flow DNS→GSLB→L4→L7→backend.

**Gaps vs plan:** ADD LB-as-SPOF/HA (VIP/VRRP/keepalived); ADD connection draining; NAME full algorithm set + decision criteria + P2C; make consistent hashing explicit; teach GSLB somewhere; ADD TLS termination; deepen health-check specifics; ADD cloud product mapping; drive home sticky-session fix = stateless.

---

### Cluster 3 — Caching (Day 3)
**Must-teach checklist**
- Patterns: cache-aside (lazy, default), **read-through**, write-through, write-back/behind, **write-around**.
- Eviction: LRU (default), LFU, FIFO, MRU, Random, + TTL.
- **TTL jitter / randomized expiry.**
- **Three failure modes (define crisply):** stampede/thundering herd (hot key expires) — fixes: per-key mutex/distributed lock, request coalescing/single-flight, probabilistic early recompute, stale-while-revalidate; **penetration** (key in neither cache nor DB) — fixes: null/negative caching short TTL, **Bloom filter**, input validation; **avalanche** (mass expiry / node death) — fixes: TTL jitter, multi-level cache, circuit breaker, throttling.
- **Hot key problem:** key replication/splitting, local L1 in front, dedicated nodes.
- Distributed topology: client-side consistent hashing vs Redis Cluster hash slots.
- Consistent hashing (cross-ref Day 9).
- **Redis vs Memcached decision criteria:** data structures, single vs multi-threaded, persistence RDB/AOF, replication/failover (Sentinel/Cluster), pub-sub.
- Consistency/coherence: stale reads, invalidation ordering, the **dual-write race** (update-DB-then-invalidate vs reverse).
- Multi-level: browser → CDN → app L1 (in-process) → distributed L2 → DB buffer pool.
- **Cache hit ratio** as effectiveness metric (worked calc, ties to Day 15: "90% hit ratio → DB QPS = …").
- Cache warming/cold start; when NOT to cache (write-heavy, low-reuse, strong-consistency data).

**Common questions:** cache-aside vs read-through vs write-through vs write-back & when; 1M hit expired key (stampede); non-existent data queries (penetration/Bloom/null); avalanche & TTL jitter; Redis vs Memcached; keep cache+DB consistent / partial-write failure; 100k req/s celebrity key (hot key); LRU vs LFU; what/TTL to cache & measure effectiveness; how key→node (consistent hashing/slots); design a distributed cache; Black Friday spike caching.

**Gaps vs plan:** ADD penetration + avalanche (trio); ADD Bloom filter + null caching; ADD hot-key mitigations; ADD read-through + write-around; NAME TTL jitter; NAME stampede techniques; make consistency/invalidation-ordering explicit; ADD multi-level + layer map; ADD hit-ratio calc; reference consistent hashing; enumerate Redis vs Memcached criteria; ADD warming + when-not-to-cache.

---

### Cluster 4 — CDN & DNS (Day 4)
**Must-teach checklist**
- Full DNS chain: stub resolver → recursive resolver (ISP/8.8.8.8/1.1.1.1) → root → TLD → authoritative NS; **recursive vs iterative**; recursive resolver vs authoritative NS.
- Caching layers + TTL: browser, OS (/etc/hosts), resolver; **positive vs negative caching (NXDOMAIN)**; resolver caching defeats GeoDNS within TTL.
- **Record types:** A, AAAA, CNAME, ALIAS/ANAME (apex problem), NS, SOA, MX, TXT, PTR, SRV, CAA.
- **Zero-downtime DNS cutover:** lower TTL 24-48h before, switch in low-TTL window, validate, raise TTL.
- **Anycast:** one IP via BGP → nearest/healthiest PoP; DNS roots, resolvers, CDN edge; DDoS absorption.
- Routing policies (Route 53): latency, weighted (canary), geolocation, geoproximity, failover, multivalue; **GSLB w/ health checks**.
- CDN fundamentals: PoP/edge/origin, **origin shield/multi-tier**, cache key, **hit ratio**, static vs dynamic.
- **Pull CDN** (lazy, long-tail/changing) vs **Push CDN** (large/predictable media) & when.
- HTTP cache control: Cache-Control (max-age, s-maxage, public/private, no-cache vs no-store), ETag/If-None-Match, stale-while-revalidate, Vary.
- **Invalidation:** URL purge vs tag/surrogate-key purge vs versioned/fingerprinted URLs (main.abc123.js); freshness vs hit-ratio.
- **Edge computing:** Cloudflare Workers / Lambda@Edge / Fastly Compute; V8 isolates (sub-1ms, no cold start) vs container/Lambda cold starts (100-1000ms); edge=stateless latency-sensitive, origin=stateful.
- CDN security: signed URLs/cookies/token auth, TLS termination at edge, WAF, DDoS, bot mitigation.
- Multi-CDN + egress cost (senior).

**Common questions:** what happens when you type a URL (DNS chain); zero-downtime region migration via DNS/TTL; design a CDN, edge cache-vs-origin decision; pull vs push for video vs news; invalidate across hundreds of edges / why versioned URLs; Anycast vs GeoDNS; DNS-based failover/GSLB; serve private/paywalled content via public CDN; why old server still hit (TTL/caching); edge vs origin & constraints; design DNS cache (LLD: TTL eviction, negative caching).

**Gaps vs plan:** make hierarchy + recursive/iterative explicit; ADD record types (CNAME-at-apex); ADD caching layers + negative caching; ADD edge computing; make invalidation explicit (purge vs tag vs versioned); ADD Cache-Control headers (bridge Day 3); ADD GSLB/Route 53 policies + health checks (cross-ref Day 2); ADD CDN security/signed URLs; ADD origin shield + hit ratio; FLAG multi-CDN + egress for Phase 3.

---

### Cluster 5 — SQL & ACID (Day 5)
**Must-teach checklist**
- ACID precise: Atomicity (rollback), Consistency (app invariants + constraints), Isolation, Durability (fsync/WAL).
- 4 isolation levels ↔ phenomena prevented: Read Uncommitted, Read Committed, Repeatable Read, Serializable.
- 3 read phenomena: dirty read, non-repeatable read, phantom read (canonical table).
- **Lost update + write skew** (DDIA) — beyond the 3; write skew needs true serializability (counter increment = lost update; on-call rota / double-booking = write skew).
- **MVCC:** multi-version + txn IDs/snapshots; readers don't block writers; how Postgres/MySQL implement RC & RR.
- **Snapshot Isolation naming trap:** Oracle/PG historical "Serializable" ≈ SI, still allows write skew.
- **WAL:** log-before-data, sequential writes, durability + crash recovery; redo vs undo.
- Pessimistic (`SELECT … FOR UPDATE`, shared/exclusive, row/table) vs optimistic (version/timestamp, compare-and-set, retry).
- **2PL** = serializability mechanism (acquire/release phases); NOT 2PC.
- Deadlocks: cyclic wait, detection (wait-for graph), resolution (victim rollback), prevention (consistent lock ordering, short txns).
- **Default isolation per engine:** PG=Read Committed, MySQL/InnoDB=Repeatable Read, Oracle/SQL Server=Read Committed.
- SSI (PG SERIALIZABLE ≥9.1): serializability ≈ SI perf via write-skew dependency detection.
- When ACID is right vs NoSQL; cross-shard txns expensive → 2PC/Saga (forward-ref).
- Connection pooling (PgBouncer) — one-liner.

**Common questions:** ACID + bank-transfer example; 4 levels & phenomena; dirty vs non-repeatable vs phantom; optimistic vs pessimistic & when; prevent double-booking/oversell (lost update); MVCC; WAL & durability; deadlock cause/prevent; serializability vs snapshot isolation & write skew; default level PG/MySQL; safe counter/wallet increment; cross-shard/microservice txns (→2PC/saga); version column vs FOR UPDATE; why not Serializable everywhere.

**Gaps vs plan:** ADD lost update + write skew; make MVCC explicit (mechanism vs contract); ADD snapshot-isolation naming trap; ADD per-engine default table; ADD 2PL vs 2PC note; forward-ref distributed txns (2PC/Saga → new day); show concrete locking SQL in task; cross-ref idempotency (Day 12) for safe-retry/exactly-once; one-liner connection pooling.

---

### Cluster 6 — NoSQL (Day 6)
**Must-teach checklist**
- **5 families** + flagship + use case: Key-Value (Redis, DynamoDB) sessions/cache; Document (MongoDB) nested objects; Wide-Column (Cassandra, HBase, ScyllaDB, Bigtable) write throughput; Graph (Neo4j) traversal; **Time-Series (InfluxDB, TimescaleDB)** metrics/IoT.
- **Vector DBs** (Pinecone, Weaviate, Milvus, pgvector): embeddings, ANN/similarity, RAG — top 2025-26 topic.
- **Query-first / access-pattern-driven modeling:** list every query first, then design keys (inverse of SQL normalization) — headline skill.
- Denormalization/duplication as deliberate (no joins in Cassandra).
- Embed vs reference (document stores): embed if read/changed together; reference for large/independently-changing.
- **Partition/hash key design + hot-partition:** high cardinality, even distribution; composite PK+SK (DynamoDB), partition + clustering columns (Cassandra).
- **BASE** vs ACID; **tunable consistency** (Cassandra QUORUM/read-repair; DynamoDB eventual vs strong reads).
- SQL-vs-NoSQL framework (myth-bust: tuned Postgres handles millions/day; decide on data shape, access patterns, query complexity, consistency, write throughput).
- Polyglot persistence (Postgres + Redis + Elasticsearch + S3).
- Secondary indexes & limits: DynamoDB **GSI vs LSI**, Cassandra secondary-index pitfalls, inverted-index pattern.
- **DynamoDB single-table design:** overloaded PK/SK, item collections, 1:many & many:many without joins.
- Materialized views / CQRS read models.
- **NewSQL / distributed SQL** (Spanner, CockroachDB, Vitess): horizontal scale + ACID.

**Common questions:** SQL vs NoSQL for this system (justify by access patterns); NoSQL types + use case each; BASE/eventual vs ACID & UX impact; model feature in DynamoDB/Cassandra (PK/SK); embed vs reference in Mongo; avoid hot partition; Cassandra no-joins query; uncovered query pattern (GSI vs 2nd table vs materialized view); design semantic search/RAG; polyglot store selection end-to-end; tunable consistency; denormalization sync trade-offs.

**Gaps vs plan:** UPDATE 4→5 families (add time-series); ADD vector DBs (+ Phase 3 RAG design day); NAME query-first modeling as headline; ADD concrete key design (PK/SK, partition+clustering) + embed/reference; ADD single-table design; ADD tunable consistency (reinforce Day 7/8); ADD GSI/LSI (or Day 10); ADD NewSQL note; ADD SQL-vs-NoSQL myth-busting; NAME materialized views/CQRS.

---

### Cluster 7 — CAP + PACELC (Day 7)
**Must-teach checklist**
- CAP: C/A/P defs; key insight = partitions unavoidable → real choice is **C vs A during a partition only**.
- **"CA" doesn't exist** for distributed systems (cannot opt out of P) — debunk explicitly.
- **PACELC:** if Partition→A vs C; Else→Latency vs C (consistency costs latency even with NO partition — the senior signal).
- PACELC classification: DynamoDB PA/EL, Cassandra PA/EL (tunable), Spanner PC/EC, RDBMS PC/EC; MongoDB = config-dependent: classic PACELC tables (Abadi 2012) list it PA/EC, but with `w:majority` writes (implicit default since 5.0, except some arbiter setups → `w:1`) + primary reads it behaves PC/EC; secondary reads push it toward EL.
- Strong (linearizability); eventual (BASE); causal; **session/client-centric: read-your-writes, monotonic reads, monotonic writes, writes-follow-reads** (all four by name + example).
- Quorum N/W/R; **W+R>N** overlap rule (teach mechanics on Day 8, reference here).
- Tunable consistency: Cassandra ONE/QUORUM/ALL/LOCAL_QUORUM/EACH_QUORUM; DynamoDB eventual vs ConsistentRead.
- **Linearizability vs serializability** (single-object recency vs multi-object txn isolation) — classic trap.
- Conflict resolution: LWW, vector clocks, version vectors, CRDTs, read-repair, anti-entropy, hinted handoff.
- Map consistency to product: money/inventory/bookings→CP; feeds/likes/profiles→AP.
- Consensus (Paxos/Raft) as the mechanism behind CP (one paragraph; deep dive later).
- CAP's "C" (linearizability) vs ACID's "C" (constraints) — tie to Day 5.

**Common questions:** explain CAP, CP or AP for THIS system & why; PACELC + classify Dynamo/Cassandra/Spanner; is there really a CA system; strong vs eventual + example each; read-your-writes & why expected; quorum N=3 W/R for strong, what if W+R≤N; tune Cassandra/Dynamo for read- vs write-heavy; linearizability vs serializability; AP conflict resolution; banking ledger vs social feed consistency; why eventual ok for likes not balance; ACID-C vs CAP-C.

**Gaps vs plan:** DROP/debunk CA as co-equal; ensure all 4 session guarantees taught; move quorum mechanics to Day 8 + reference; ADD linearizability vs serializability; STRESS PACELC Else-branch; ADD conflict-resolution mechanisms (LWW/vector clocks/CRDTs); make PACELC classification a deliverable; ADD Raft/Paxos one-paragraph; tie CAP-C vs ACID-C to Day 5.

---

### Cluster 8 — Replication (Day 8)
**Must-teach checklist**
- 3 topologies: single-leader (master-slave), multi-leader (master-master), **leaderless (Dynamo-style)** & when each.
- Sync vs async vs **semi-sync** (leader waits ≥1 follower ack); durability/latency/availability triangle.
- Replication lag → async = eventual consistency.
- **DDIA lag-anomaly trio + fixes:** read-after-write (route self-writes to leader / by timestamp), monotonic reads (pin user to one replica via hash), consistent prefix reads (preserve causal order).
- Read replicas / read scaling + stale-read downside.
- **Failover (single-leader):** detect failure, promote follower, choose most up-to-date replica; dangers = lost writes (async), split-brain.
- **Split-brain** → fencing/quorum.
- **Leaderless quorum:** N/W/R, **W+R>N**; W=N,R=1 fast reads; W=1,R=N fast writes; W=R=QUORUM balanced.
- Read repair + anti-entropy; hinted handoff.
- Conflict resolution: LWW (clock-skew data-loss risk), vector clocks/version vectors, CRDTs (G-Counter, PN-Counter, OR-Set), app-level merge.
- Replication methods: statement-based vs WAL/physical vs logical/row-based (ties to Day 5 WAL).
- **Chain replication** (head=writes, tail=reads, strong consistency) — brief awareness only.
- Real-system map: PG/MySQL single-leader + replicas + semi-sync; Galera/Group Replication multi-leader; Cassandra/Dynamo/Riak leaderless. Cross-ref Day 7 (CAP) & Day 9 (replication=copies vs sharding=split).

**Common questions:** single vs multi vs leaderless & when; sync vs async vs semi-sync trade-offs; replication lag problems + read-after-write fix; profile edit then stale reload — why & fix; scale reads + downside; quorum W+R>N (N=3 strong; W=R=1?); leaderless sync (read repair/anti-entropy/hinted handoff); conflicting writes (LWW vs vector clocks vs CRDTs); split-brain & failover; what goes wrong in failover; global multi-region multi-leader + conflicts; chain replication.

**Gaps vs plan:** ADD leaderless + quorum math; ADD lag-anomaly trio + fixes; ADD conflict resolution; ADD failover + split-brain; ADD semi-sync; ADD leaderless repair mechanisms; brief chain replication only; ADD replication-method types (link Day 5 WAL); cross-link Day 7 & 9.

---

### Cluster 9 — Sharding & Partitioning (Day 9)
**Must-teach checklist**
- Horizontal (sharding) vs vertical partitioning vs normalization; sharding ≠ replication.
- Strategies: **range**, **hash (modulo)**, **directory/lookup** — pros/cons.
- Modulo flaw: `hash(key)%N` remaps ~all keys when N changes → motivates consistent hashing.
- **Consistent hashing:** hash ring, clockwise assignment, only ~k/n keys move.
- **Virtual nodes:** one token/node = uneven; vnodes smooth load + faster rebalance (Cassandra, DynamoDB).
- **Shard-key selection:** high cardinality, even distribution, query alignment; bad keys (boolean is_premium, monotonic timestamp/auto-increment = write hotspot).
- **Hotspot/celebrity:** access skew, not data skew; **vnodes do NOT fix a single hot key**. Mitigations: dedicated shard, key splitting/salting, caching, read replicas for hot shard, compound keys.
- Rebalancing (DDIA): fixed partition count (many > nodes, steal partitions) vs dynamic split/merge (HBase/MongoDB chunks); move minimal data, nonblocking.
- Cross-shard queries / **scatter-gather** fan-out latency amplification (global top-10 across N shards).
- Cross-shard txns: lose single-node ACID → 2PC (blocking/coordinator failure) vs Saga; design to co-locate related data.
- Secondary indexes under sharding: local/document-partitioned (scatter-gather reads) vs global/term-partitioned (cross-ref Day 10).
- Online resharding: double-write + backfill.
- **When NOT to shard:** premature anti-pattern — exhaust vertical → replicas → cache → shard (with Day 15 math).
- Real systems: Cassandra/DynamoDB (consistent hashing + vnodes), MongoDB (range chunks + balancer), Vitess/Citus (sharded SQL).

**Common questions:** sharding vs partitioning vs replication; range vs hash vs directory; why `hash%N` bad; consistent hashing + keys moved; vnodes; choose shard key for users/orders/tweets; celebrity melts a shard (why vnodes won't fix); top-10 across 64 shards; cross-shard txn / 2PC downsides; add capacity w/o downtime; secondary indexes when sharded; timestamp shard write hotspot; shard vs replicas/cache decision.

**Gaps vs plan:** ADD consistent hashing (ring+vnodes) explicitly; NAME 3 strategies; ADD shard-key criteria + bad-key examples; clarify hot-key access-skew nuance + mitigation menu; ADD rebalancing; ADD cross-shard joins/txns + scatter-gather + 2PC/Saga; ADD secondary-index partitioning (link Day 10); ADD when-NOT-to-shard ladder; ADD online resharding; ADD real-system anchors.

---

### Cluster 10 — Indexing (Day 10)
**Must-teach checklist**
- Index = separate sorted structure; trades write cost + storage for read speed.
- **B-Tree/B+Tree:** balanced, O(log n), in-place updates, page splits, equality+range, sorted ORDER BY; default PG/MySQL/InnoDB.
- **LSM-Tree:** memtable → immutable SSTables, sequential writes, background compaction; Cassandra/RocksDB/LevelDB/ScyllaDB/Bigtable.
- LSM supporting structures: **WAL**, **Bloom filters**, SSTable, compaction (size-tiered vs leveled).
- **Write amplification vs read amplification** (+ space amplification).
- B-Tree vs LSM decision: read-heavy/point/range/strong-consistency → B-Tree; write-heavy/high-ingest (logs/metrics/time-series) → LSM.
- **Clustered vs non-clustered (secondary)** index; secondary points back to PK (extra lookup), non-unique, maintained per write.
- Primary vs secondary.
- **Composite index + leftmost-prefix rule:** (state,zip) serves state or (state,zip) but NOT zip alone; column order matters.
- **Covering index** (INCLUDE / index-only scan).
- **Inverted index:** term→doc IDs; full-text (Elasticsearch/Lucene/Solr); tokenization, postings.
- **Hash index:** O(1) equality, no range/order.
- **Geospatial/spatial:** geohash, quadtree, R-tree, S2, H3 (Uber/Yelp/maps).
- When indexes hurt: write-heavy, unused/redundant, small tables, storage cost, slower writes, write amplification.
- Selectivity/cardinality (low-cardinality = poor candidate; planner may ignore).
- Local vs global secondary index under sharding (link Day 9).

**Common questions:** what is an index + trade-off; B-Tree vs LSM & when; why LSM for writes + write amplification; Bloom filter speeding LSM reads; composite index leftmost-prefix; covering index; clustered vs non-clustered; full-text/inverted index; proximity service geohash vs quadtree; when index hurts; slow query → which index / read EXPLAIN; secondary index in sharded DB (local vs global); why Cassandra LSM vs Postgres B-tree.

**Gaps vs plan:** ADD LSM internals (memtable→SSTable→compaction, WAL, Bloom); NAME write vs read amplification; ADD leftmost-prefix rule; ADD covering index; ADD clustered vs non-clustered; ADD geospatial (or forward-ref Phase 3); ADD hash index; ADD selectivity/cardinality; ADD local vs global secondary index; TASK should interpret an EXPLAIN plan.

---

### Cluster 11 — Message Queues & Streaming (Day 11)
**Must-teach checklist**
- **Queue vs Stream vs Pub/Sub:** point-to-point work queue (one consumer/msg; RabbitMQ/SQS) vs log-based stream w/ replay + multiple consumer groups (Kafka).
- Three techs as different architectures: Kafka = distributed append-only log w/ partitions; RabbitMQ = smart broker (exchanges: direct/topic/fanout/headers); SQS = managed HTTP queue (standard vs FIFO).
- **Delivery semantics:** at-most-once, at-least-once (default), **exactly-once = at-least-once + dedup ("effectively once")**; true exactly-once impossible (Two Generals).
- **Idempotent consumer:** idempotency keys, dedup store of processed IDs, DB unique constraints/upserts.
- Kafka internals: topics, partitions, offsets, consumer groups (1 partition→1 consumer/group), `partition=hash(key)%n`, rebalancing.
- Ordering only within a partition; global order = single partition (kills parallelism).
- Hot/skewed partition + fixes (key choice, salting, compound keys, backpressure).
- Durability: leader/follower replicas, ISR, **acks=0/1/all**, replication factor 3, min.insync.replicas.
- Offset commit timing (after vs before processing → at-least vs at-most-once; auto vs manual).
- **DLQ:** poison messages, max-retry, prevents head-of-line blocking, replay.
- Retry: transient-only, exponential backoff + jitter, max attempts, retryable vs not.
- Backpressure: queue growth, pull-based, bounded buffers, autoscale on lag/depth, throttling; Kafka log as buffer.
- **Consumer lag** = key monitoring metric.
- Retention/replay (retention.ms/bytes); log compaction (latest value per key).
- **Transactional Outbox + CDC** (dual-write problem).
- Event sourcing (immutable append-only log + snapshots) — move to Phase 2.
- CQRS (separate write/read models, eventual consistency) — move to Phase 2.
- Fan-out (on-write vs on-read); pub/sub multiple subscribers.
- **Claim-check anti-pattern:** no blobs in messages → S3 + reference.

**Common questions:** Kafka vs RabbitMQ vs SQS & when; guarantee exactly-once (you can't — at-least-once + idempotent); needed delivery guarantee & how; Kafka ordering + adding partitions; hot partition fix; poison message (DLQ + backoff); queue backing up (backpressure/autoscale/lag); idempotent consumer; don't lose messages on broker crash (acks=all/ISR); event sourcing + CQRS & downsides; atomic DB-write + publish (outbox/CDC); design distributed queue/fan-out; push vs pull; consumer lag monitoring; producer partition selection.

**Gaps vs plan:** ADD delivery semantics + idempotent consumer; ADD Kafka mechanics (partitions/groups/offsets/ordering/hot partition); ADD DLQ + retry/backoff; ADD backpressure + lag; ADD durability (acks/ISR/replication); ADD Outbox + CDC; MOVE event sourcing + CQRS to Phase 2; cross-link idempotency to Day 12; ADD queue-vs-stream-vs-pubsub framework + claim-check; consider "design a distributed queue" in Phase 3.

---

### Cluster 12 — APIs & Gateway (Day 12)
**Must-teach checklist**
- **REST vs GraphQL vs gRPC matrix:** REST default for public/cacheable resources; GraphQL client-driven shapes, single endpoint, solves over/under-fetching; gRPC HTTP/2 + Protobuf binary, multiplexing, internal service-to-service polyglot.
- When NOT GraphQL/gRPC: GraphQL **N+1 (needs DataLoader batching)**, harder caching, field-level authz; gRPC poor for browsers, harder debug. Default REST unless justified.
- **Idempotency:** idempotent verbs (GET/PUT/DELETE/HEAD/OPTIONS yes; POST no; PATCH depends). **Idempotency-key pattern:** client generates key in header, server stores first result + replays on retry (payments double-charge example).
- **Pagination:** offset/limit (simple, jump-to-page, but slow at scale + skips/dupes on shift) vs **cursor/keyset** (stable, scales, no jump).
- **Versioning:** URL path (/v1/, preferred), header/Accept media-type, query param; APIs are contracts.
- HTTP status codes + structured error bodies: 2xx (200/201/204), 4xx (400/401/403/404/**409**/**429**), 5xx.
- authn vs authz: API keys (server-server), JWT (stateless sessions), OAuth2 (delegated), RBAC.
- **API Gateway responsibilities:** single entry, routing, authn/authz, rate limiting, TLS termination, transformation, aggregation, logging, protocol translation (REST→gRPC).
- **BFF pattern:** per-client backend (web/iOS/Android) shaping data; distinct from gateway (one for all); BFF often behind gateway.
- **Webhooks (reverse API):** HMAC signature verification + timestamp tolerance (Stripe), at-least-once, idempotent consumers, exponential-backoff retries, DLQ; timeout-induced duplicate = classic failure.
- Resource modeling: nouns not verbs; path params required, query params optional.
- Rate limiting as gateway concern: 429 + Retry-After (→ Day 14).

**Common questions:** design Ticketmaster booking API (endpoints, high-contention); make POST safe vs retries (idempotency keys); REST vs GraphQL vs gRPC & why; mobile vs web data (GraphQL vs BFF); paginate millions (offset vs cursor); version w/o breaking clients; design webhook delivery (reliability/retries/security); gateway vs LB vs reverse proxy vs BFF; status code + error body for X; authn/authz at gateway; GraphQL N+1 fix.

**Gaps vs plan:** ADD pagination (cursor vs offset); ADD idempotency-KEY mechanism (not just verbs); ADD versioning strategies; ADD webhook reliability engineering; ADD BFF + distinction from gateway; ADD status-code taxonomy + error model; ADD auth (API keys/JWT/OAuth2/RBAC) or flag for Security day; ADD GraphQL N+1/DataLoader; deconflict gateway (12) vs reverse proxy (13) vs rate limiting (14); ADD resource modeling. **Also add WebSocket vs SSE vs long-polling here** (blocks chat/notifications/Uber/Drive).

---

### Cluster 13 — Proxies & Reverse Proxies (Day 13)
**Must-teach checklist**
- **Forward proxy** (fronts CLIENTS: egress, anonymity, content filtering, corporate firewall, outbound caching) vs **reverse proxy** (fronts SERVERS: ingress facade) — most common confusion.
- Reverse proxy jobs: hide/protect backend topology, TLS termination, static caching, compression (gzip/brotli), path routing, header rewriting.
- **Reverse proxy vs LB:** every LB is a reverse proxy, not every reverse proxy is a LB (RP can front a single server); production uses both.
- **TLS termination** at proxy (offload CPU, central cert mgmt, L7 inspection); passthrough vs termination vs re-encryption (bridging) for zero-trust.
- Nginx: event-driven/async (vs Apache thread-per-conn); reverse proxy + L7 LB; upstream blocks, proxy_pass, round-robin/least_conn/ip_hash, health checks; Netflix/Dropbox.
- API gateway = specialized app-aware reverse proxy (authn/authz, rate limit, transformation, composition, versioning, observability) — overlaps Day 12.
- **Sidecar pattern:** helper container co-deployed (shared network namespace), cross-cutting concerns, app talks to localhost, app-unaware.
- **Service mesh:** data plane (Envoy sidecars) + control plane (Istiod); traffic mgmt, mTLS, retries, timeouts, circuit breaking, observability, traffic splitting/canary.
- **Envoy** = de-facto L7 sidecar data plane.
- **mTLS / zero-trust** service-to-service auth.
- **North-south** (client↔system: LB/gateway/RP) vs **east-west** (service↔service: mesh) traffic.
- **K8s Ingress + Ingress Controller** (Nginx Ingress) = cloud-native reverse proxy/L7 routing.
- **When NOT to use a service mesh:** added latency hop, per-sidecar CPU/RAM (Istio 1.24 benchmark ≈ 0.2 vCPU / 60MB at 1k req/s, grows with traffic; ambient mode cuts this), ops complexity — only at meaningful microservice scale.
- META source map: primer (breadth/glossary), Alex Xu Vol1 (framework + 16 fundamentals) / Vol2 (13 advanced designs), DDIA (data-layer depth, no proxies/mesh), Grokking (building blocks → case studies).

**Common questions:** forward vs reverse proxy + example; reverse proxy vs LB & need both; API gateway vs RP/LB; TLS termination & why at proxy; why Nginx in front; what is a sidecar & what it solves; service mesh data vs control plane; when adopt mesh + downsides; mesh secures service-to-service (mTLS); north-south vs east-west; K8s external→service (Ingress); RP improves security/performance.

**Gaps vs plan:** ADD forward proxy use cases; ADD RP-vs-LB explicit (cross-link Day 2); ADD TLS termination (+ Day 2/4 refs); ADD gateway/RP/LB 3-way table (span Day 12/13); teach sidecar as first-class; NAME Envoy + Istio/Linkerd + data/control plane; ADD mTLS/zero-trust; ADD north-south vs east-west; ADD K8s Ingress; ADD when-NOT-mesh to eval; ADD source-map orientation (Day 1 or 15).

---

### Cluster 14 — Rate Limiting (Day 14)
**Must-teach checklist**
- **Token bucket:** bucket size (burst) + refill rate; allows bursts, enforces average (Stripe/AWS).
- **Leaky bucket:** FIFO queue, fixed outflow, smooths bursts.
- **Fixed window counter:** INCR+EXPIRE; simplest but **boundary spike** (up to 2x at edges).
- **Sliding window log:** sorted set of timestamps, ZREMRANGEBYSCORE; accurate but high memory.
- **Sliding window counter:** weight prev window by overlap; near-accurate, low memory — the common "right answer."
- Placement: in-process/middleware (fast, no global view) vs dedicated service (centralized, +latency hop) vs **API Gateway (preferred)**; client- vs server-side.
- **Distributed coordination:** N nodes must share counter or each allows full limit (×N).
- **Read-modify-write race:** two instances read 99, both allow → 101 slips.
- **Atomicity fix:** Redis atomic INCR or **Lua script** (check-and-increment in one op; cuts round trips).
- Redis shared store; shard/consistent-hash keys (same client→same shard); hot-key when one client dominates.
- HTTP: **429 Too Many Requests** + Retry-After + X-RateLimit-Limit/Remaining/Reset.
- **Fail-open vs fail-closed** when limiter/Redis down (fail-open = general APIs; fail-closed = login/security).
- Identity dimensions: user ID, API key, IP, endpoint; tiered per-plan limits.
- Latency/accuracy/memory trade-off; "understand ONE algorithm deeply."

**Common questions:** design rate limiter for millions RPS; which algorithm & why (token bucket vs sliding window counter); where in architecture; work across distributed servers; two instances same user (race/atomicity); why Lua script vs GET-then-INCR; Redis down — fail open or closed & why endpoint-dependent; status + headers when throttled; hot key/abusive client; fixed-window boundary problem; per-tier/API-key/IP limits; estimate memory for N million users.

**Gaps vs plan:** ADD 4-way algorithm breakdown (name sliding window counter + log + fixed window boundary); ADD placement (client/middleware/gateway/service); ADD distributed RL + race + Redis INCR/Lua (HIGH priority); ADD 429 + headers; ADD fail-open vs fail-closed; ADD key sharding + hot key; ADD key dimensions + tiers; ADD memory estimate to Day 15; OPTIONAL mention AIMD/adaptive (Phase 3).

---

### Cluster 15 — Back-of-Envelope Estimation (Day 15)
**Must-teach checklist**
- **Jeff Dean latency numbers:** L1 0.5ns, L2 7ns, mutex 25ns, memory ref 100ns, 1KB Zippy 3µs, 1KB over 1Gbps 10µs, DC round trip 500µs, read 1MB seq SSD ~1ms, HDD seek 10ms, read 1MB HDD 20-30ms, CA↔Netherlands round trip 150ms.
- Relative intuition > memorization: memory ~1,000x SSD for random access (100ns vs 150µs) but only ~4x for 1MB sequential, SSD ~20x HDD sequential (more for random), same-DC cheap, cross-region WAN dominates.
- Powers of two: 2^10=KB, 2^20=MB, 2^30=GB, 2^40=TB, 2^50=PB; powers of ten for counts.
- **QPS = (DAU × actions/user/day) / 86,400**; shortcut "1M/day ≈ 12 QPS avg."
- **Peak QPS = avg × 2-10x** (state factor + why bursty).
- Read/write ratio estimated separately (e.g. 100:1).
- DAU vs MAU + stickiness ratio.
- **Storage = bytes/item × items/day × retention × replication** + metadata + growth (5yr).
- Bandwidth = bytes/req × QPS; bytes↔bits (×8) → Mbps/Gbps.
- Memory/cache sizing: **80/20 rule**, cache QPS, RAM, hit-ratio impact.
- Server count = total load / per-server capacity.
- **Availability nines table:** 99%=3.65d/yr, 99.9%≈8.76h, 99.99%≈52.6min, 99.999%≈5.26min; dependent services multiply.
- **4-step framework:** clarify assumptions → approximate aggressively → break down/aggregate → sanity-check.
- Data-size rules: char ~1B, long/timestamp 8B, tweet/text ~few KB, photo ~200KB-2MB, 1hr HD video ~1GB.
- Latency composition: sequential ADD, parallel MAX; **tail p99 vs average**.

**Common questions:** Twitter QPS (300M DAU, 2 tweets/day → avg & peak write QPS, read QPS at 100:1); 5-yr storage w/ replication (Instagram/photo); bandwidth for YouTube/Netflix at peak; cache memory via 80/20; server count for peak QPS; rank memory/SSD/disk/network latency; 99.99% downtime/yr implication; DAU from user base + URL shortener sizing; why estimation matters / which numbers worth calculating; 10x peak recompute.

**Gaps vs plan:** ADD Jeff Dean table + relative intuition; ADD nines table + multiply rule; ADD peak multiplier reasoning; NAME read/write ratio input; ADD DAU/MAU stickiness; ADD 80/20; make powers-of-two a memorization target; ADD latency composition + tail latency; make 4-step framework the spine; ADD data-size reference card; TASK = full worked example (DAU→QPS→peak→storage→bandwidth→cache→servers) + warn against false precision.

---

### Cluster 16 — Distributed Primitives (NEW DAY)
**Must-teach checklist**
- Consensus + why; split-brain (two leaders) is the danger.
- Quorum/majority: write/election needs (N/2)+1; R+W>N overlap.
- **Raft (interview default):** randomized election timeouts (150-300ms), terms, RequestVote/AppendEntries, log replication, commit on majority; understandability vs Paxos.
- Paxos (name-drop, original proven-correct, hard), Multi-Paxos, ZAB (ZooKeeper).
- Real systems: etcd/Raft (K8s), ZooKeeper/ZAB (Kafka), Consul/Raft (service discovery) — use these, don't build your own.
- **Distributed locks:** single-node insufficient; Redis SETNX+TTL/Redlock + criticisms; ZooKeeper/etcd ephemeral nodes safer.
- **Fencing tokens:** monotonic token rejected by resource if stale — canonical correctness fix.
- Lease vs lock; clock skew; paused process holding "expired" lease.
- **Logical clocks:** Lamport (happens-before) vs vector clocks (concurrency/causality; Dynamo/Cassandra).
- LWW vs causal; why wall-clock timestamps unsafe across nodes (skew, NTP).
- **Idempotency keys:** client UUID per logical op (not per HTTP retry); server (key→result) store, Redis TTL + durable table.
- Exactly-once = at-least-once + idempotent processing.
- **2PC** (prepare/commit, coordinator blocking, locks across phases) vs **Saga** (local txns + compensating; choreography vs orchestration).
- **Outbox pattern:** business change + event in same DB txn → relay; avoids dual-write w/o 2PC.
- Heartbeats + failure detection; phi-accrual; can't distinguish slow from dead.
- **Gossip / SWIM** membership (Cassandra/Dynamo/Consul); anti-entropy.

**Common questions:** elect a leader / prevent split-brain in partition; Raft election + log replication; design distributed lock + GC-pause/TTL problem + fencing fix; is Redlock safe; idempotency keys for payment charge; design Stripe exactly-once charging; consistency across 2 microservice DBs w/o distributed txn (Saga+outbox); 2PC vs Saga + compensating txn; node failure detection (heartbeats/phi/gossip); Cassandra/Dynamo conflict detection (vector clocks); why not wall-clock timestamps (logical clocks); cluster membership w/o central coordinator (gossip); quorum R+W>N.

**Gaps vs plan:** entire cluster absent — add dedicated day(s) covering all above; cross-link Day 5 (locking), Day 7/8 (clocks/quorum), Day 11 (exactly-once), Day 12 (idempotency).

---

### Cluster 17 — Ops & Security (NEW DAYS — split into Observability, Resilience, Security, Deployment)
**Must-teach checklist**
- **Observability:** 3 pillars (logs/metrics/traces); monitoring vs observability; **Golden Signals** (latency/traffic/errors/saturation) + **RED** (Rate/Errors/Duration) + **USE** (Utilization/Saturation/Errors); structured logging + correlation/request IDs; distributed tracing (trace+span ID, context propagation, head- vs tail-based sampling, Jaeger/Zipkin/OpenTelemetry); metrics stack (Prometheus pull-based, Grafana, alerting); **SLI/SLO/SLA + error budgets + burn-rate alerts**; liveness vs readiness probes (deep vs shallow).
- **Resilience:** circuit breaker (closed/open/half-open, failure-rate/latency thresholds, fail-fast, fallback); **bulkhead** (isolate thread/conn pools); **retries + exponential backoff + full jitter + retry budgets** (combine w/ idempotency); timeouts + deadline propagation; graceful degradation + graceful shutdown (drain, stale fallback, load shed); cascading failure prevention; chaos engineering.
- **Security:** authn vs authz; session vs token (JWT) + revocation problem; **OAuth2/OIDC** roles (resource owner/client/auth server/resource server), access vs refresh tokens; JWT structure (header.payload.signature), stateless validation, expiry, hard to revoke; **RBAC vs ABAC**, least privilege; TLS 1.3 in transit, mTLS service-to-service/zero-trust; encryption at rest, secrets mgmt (Vault/KMS), password hashing (bcrypt/scrypt/Argon2, never plaintext); gateway authn/rate-limit/validation/TLS; DDoS, OWASP Top 10, input validation, PII/privacy, audit logging.
- **Deployment:** blue-green (instant cutover + rollback), canary (gradual %), rolling update + trade-offs; **feature flags** (decouple deploy from release, kill switch); monolith vs microservices trade-offs; service mesh (Istio/Envoy).

**Common questions:** add observability (what to log/metric/trace, trace across services); slow downstream w/o cascading failure; client retries — avoid double-charge; liveness vs readiness + LB use; circuit breaker 3 states; zero-downtime deploy (blue-green vs canary vs rolling); session cookies vs JWT + revoke JWT; OAuth2 flow + access vs refresh token; service-to-service auth (mTLS/zero-trust); store passwords securely; SLI vs SLO vs SLA + burn-rate alert; monolith over microservices when; prevent thundering herd/retry storm; data in transit + at rest + secrets; fast rollback.

**Gaps vs plan:** entire cluster largely absent — add **Observability day**, **Resilience day**, **Security & Auth day**, **Deployment Strategies day**; reinforce idempotency in resilience; ADD SLI/SLO/SLA + error budgets; make monolith-vs-microservices explicit (Phase 2); ADD health checks (Day 2 or resilience); mention chaos engineering + graceful shutdown.

---

### Cluster 18 — Real System Designs (Phase 3) — see §4

---

## 3. Proposed Curriculum Adjustments

### Edits to existing Days 1-15 (what to ADD to each brief)
- **Day 1 Scalability:** + availability nines table + series/parallel math; + latency vs throughput + perf-vs-scalability defs + tail p99; + object storage (block/file, tiers, URL-in-DB, presigned URLs, CDN pairing); + state-externalization mechanics; + redundancy/failover (active-passive/active-active/N+1); + read/write-ratio lens; + autoscaling/elasticity; (optional) AKF cube; + source-map orientation note.
- **Day 2 Load Balancers:** + LB-as-SPOF/HA (VIP, VRRP, keepalived, active-active/passive); + connection draining/deregistration delay; + full algorithm list + decision criteria + P2C; + consistent hashing (ring+vnodes) in LB context; + GSLB (or defer to Day 4 boundary); + TLS termination (L7 only); + deeper health checks; + cloud mapping (ALB=L7/NLB=L4, HAProxy/Envoy/F5); + sticky-fix = stateless.
- **Day 3 Caching:** + penetration + avalanche (complete the trio); + Bloom filter + null caching; + hot-key mitigations; + read-through + write-around; + TTL jitter; + named stampede techniques; + consistency/invalidation ordering (dual-write race); + multi-level L1/L2 + layer map; + hit-ratio calc; + Redis-vs-Memcached decision criteria; + warming + when-not-to-cache; reference consistent hashing.
- **Day 4 CDN & DNS:** + explicit root→TLD→authoritative hierarchy + recursive vs iterative; + record types (CNAME-apex); + caching layers + negative caching; + edge computing (Workers/Lambda@Edge/Fastly, V8 isolates vs cold starts); + invalidation (purge vs tag vs versioned URLs); + Cache-Control headers; + GSLB/Route 53 policies + health checks; + CDN security/signed URLs; + origin shield + hit ratio; flag multi-CDN/egress for Phase 3.
- **Day 5 SQL & ACID:** + lost update + write skew; + MVCC as mechanism; + snapshot-isolation naming trap; + per-engine default-level table; + 2PL vs 2PC note + forward-ref distributed txns; + concrete locking SQL in task; + cross-ref idempotency (Day 12); + connection pooling one-liner. **+ distributed locks/fencing tokens** (or in new Distributed Primitives day).
- **Day 6 NoSQL:** 4→**5 families** (+ time-series); + vector DBs; + query-first modeling (headline); + concrete key design (PK/SK, partition+clustering) + embed/reference; + single-table design; + tunable consistency; + GSI/LSI; + NewSQL note; + SQL-vs-NoSQL myth-bust; + materialized views/CQRS; + gossip/SWIM mention.
- **Day 7 CAP+PACELC:** debunk CA; + all 4 session guarantees; + linearizability vs serializability; stress PACELC Else-branch; + conflict resolution (LWW/vector clocks/CRDTs); + PACELC classification deliverable; + Raft/Paxos one-paragraph; + logical clocks intro; tie CAP-C vs ACID-C.
- **Day 8 Replication:** + leaderless + quorum N/W/R + W+R>N (teach mechanics here); + lag-anomaly trio + fixes; + conflict resolution; + failover + split-brain + fencing; + semi-sync; + read repair/anti-entropy/hinted handoff; + replication-method types (link WAL); brief chain replication; + CDC/dual-write mention; cross-link Day 7/9.
- **Day 9 Sharding:** + consistent hashing (ring+vnodes) explicit; + name 3 strategies; + shard-key criteria + bad-key examples; + hot-key access-skew nuance (vnodes don't fix) + mitigation menu; + rebalancing (fixed vs dynamic); + cross-shard joins/txns + scatter-gather + 2PC/Saga ref; + secondary-index partitioning (link Day 10); + when-NOT-to-shard ladder; + online resharding; + **Snowflake distributed ID generation** (why auto-increment fails sharded, 64-bit layout, clock skew); + real-system anchors.
- **Day 10 Indexing:** + LSM internals (memtable/SSTable/compaction/WAL/Bloom); + write vs read amplification; + leftmost-prefix rule; + covering index; + clustered vs non-clustered; + **geospatial (geohash/quadtree/H3/S2)**; + hash index; + selectivity/cardinality; + local vs global secondary index; + **probabilistic structures (Bloom, count-min sketch)**; TASK = read an EXPLAIN plan.
- **Day 11 Queues & Streaming:** + delivery semantics + idempotent consumer; + Kafka mechanics (partitions/groups/offsets/ordering/hot partition); + DLQ + retry/backoff; + backpressure + consumer lag; + durability (acks/ISR/replication factor); + Outbox + CDC; + queue-vs-stream-vs-pubsub framework + claim-check; MOVE event sourcing + CQRS → Phase 2; cross-link idempotency Day 12.
- **Day 12 APIs:** + pagination (cursor vs offset); + idempotency-key mechanism; + versioning strategies; + webhook reliability (HMAC/timestamp/retries/DLQ); + BFF vs gateway; + status-code taxonomy + error model; + GraphQL N+1/DataLoader; + resource modeling; + **WebSocket vs SSE vs long-polling**; deconflict w/ Day 13/14; auth → Security day.
- **Day 13 Proxies:** + forward proxy use cases; + RP-vs-LB explicit; + TLS termination (passthrough/terminate/re-encrypt); + gateway/RP/LB 3-way table; + sidecar first-class; + Envoy + Istio/Linkerd + data/control plane; + mTLS/zero-trust; + north-south vs east-west; + K8s Ingress; + when-NOT-mesh; + source-map note.
- **Day 14 Rate Limiting:** + 4-way algorithm breakdown (fixed-window boundary, sliding log, sliding counter); + placement (client/middleware/gateway/service); + distributed RL + race + Redis INCR/Lua; + 429 + headers; + fail-open vs fail-closed; + key sharding + hot key; + key dimensions + tiers.
- **Day 15 Estimation:** + Jeff Dean latency table + relative intuition; + nines table + multiply rule; + peak multiplier; + read/write ratio; + DAU/MAU; + 80/20 cache sizing; + powers-of-two memorization; + latency composition + tail latency; + 4-step framework as spine; + data-size card; + full worked example task; + rate-limiter memory estimate.

### NEW days to insert (mapped to phases)
| New day | Topic | Where | Why |
|---|---|---|---|
| **Day 16** | **Answer Framework + Functional vs Non-functional Requirements** | Opens Phase 2 | The spine of every interview; nothing teaches it now |
| **Day 17** | **Distributed Primitives I: Consensus, Raft, Quorum, Leader Election, Failure Detection, Gossip** | Phase 2 | Biggest senior-level gap |
| **Day 18** | **Distributed Primitives II: Distributed Locks + Fencing Tokens, Logical Clocks, Idempotency Keys, Exactly-once** | Phase 2 | Correctness toolkit |
| **Day 19** | **Distributed Transactions: 2PC vs Saga, Outbox, CDC, dual-write** | Phase 2 | Microservices staple |
| **Day 20** | **Fan-out (push/pull/hybrid), Event Sourcing & CQRS** | Phase 2 (before Twitter design) | Central newsfeed/notification decision |
| **Day 21** | **Resilience Patterns: circuit breaker, bulkhead, retries+backoff+jitter, timeouts, graceful degradation** | Phase 2 | Cascading-failure prevention |
| **Day 22** | **Observability: 3 pillars, Golden Signals/RED/USE, SLI/SLO/SLA, error budgets, OpenTelemetry** | Phase 2 | Asked in nearly every senior interview |
| **Day 23** | **Security & Auth: authn/authz, OAuth2/OIDC, JWT, RBAC/ABAC, TLS/mTLS, password hashing, secrets, zero-trust** | Phase 2 | Glaring omission |
| **Day 24** | **Deployment & Release: blue-green, canary, rolling, feature flags, monolith vs microservices** | Phase 2 | How systems ship safely |

(Renumber subsequent Phase 2 design-pattern days accordingly; Phase 3 real designs follow.)

---

## 4. Canonical Real-Design Question Bank (Phase 3)

### The answer framework (the spine for every design)
1. **Functional requirements** (features/use cases) + **Non-functional requirements** — phrased & quantified: "highly available", "100M+ DAU", "p99 < 200ms", durability, **consistency-vs-availability choice**. Non-functional reqs DRIVE the deep dives.
2. **Core entities / data model.**
3. **API design** (the contract — endpoints, request/response).
4. *(optional)* Data flow.
5. **High-level design** — boxes & arrows satisfying each API.
6. **Deep dives** — driven by non-functional reqs (scale, consistency, latency, hot keys).
7. **Bottlenecks / failure modes / cost.**
Modern guidance: do estimation **only when it changes a decision** (e.g. fits in one Redis vs needs sharding), not as ritual.

### Standard problems + key concepts each must cover
| # | Problem | Must-cover concepts |
|---|---|---|
| 1 | **URL shortener (TinyURL/bit.ly)** | base62 vs hashing, collision handling, counter/key-gen service, 301 vs 302 (301 caches, kills analytics), read-heavy → cache+CDN, custom alias, expiry |
| 2 | **Twitter / Newsfeed** | timeline gen, fan-out-on-write vs on-read, **celebrity/hot-key**, hybrid (push normal + pull-merge celebrities), ranking, Redis timeline cache |
| 3 | **WhatsApp / Messenger / chat** | WebSocket vs SSE vs long-poll, connection/session service (user→server), ordering, delivery/read receipts, at-least-once + dedup, offline queue, presence, APNs/FCM, group fan-out |
| 4 | **Rate limiter** | token bucket vs sliding window, distributed counter + Redis Lua atomicity, 429 + headers |
| 5 | **Web crawler** | URL frontier (BFS + priority), politeness/robots.txt/per-domain rate limit, URL dedup (Bloom), content dedup, trap avoidance, distributed workers |
| 6 | **YouTube / Netflix (video)** | chunked upload via presigned URLs, transcoding pipeline (queue + workers, multi-res/codec), adaptive bitrate (HLS/DASH), CDN delivery, metadata store, thumbnails |
| 7 | **Uber / Lyft / proximity (Yelp)** | geospatial index (geohash, quadtree, Redis GEO), frequent driver location updates, matching, supply/demand, WebSocket live location |
| 8 | **Google Drive / Dropbox** | file chunking (~4MB), content-addressed storage (SHA-256), block-level dedup, delta sync, metadata DB separate from blob, presigned direct-to-S3, WebSocket sync, conflict resolution |
| 9 | **Ticketmaster / booking** | optimistic vs pessimistic locking, **distributed lock + TTL (Redis)**, seat hold/reservation w/ expiry, virtual waiting queue for flash-sale, prevent oversell, idempotent payment |
| 10 | **Notification system** | multi-channel fan-out (push/SMS/email), 3rd-party providers (APNs/FCM/SES), retries+backoff, dedup, rate limiting, user prefs, queue decoupling |
| 11 | **Search autocomplete / typeahead** | trie/prefix tree, top-K per prefix, precompute+cache top results at nodes, frequency ranking, debounce, weekly offline aggregation |
| 12 | **Distributed unique ID (Snowflake)** | why auto-increment fails sharded, 64-bit layout (timestamp\|machine\|sequence), time-sortability (B-tree inserts), clock skew; UUID/ULID alternatives |
| 13 | **Top-K / trending** | count-min sketch / heavy hitters, streaming aggregation, approximate counting at scale |
| 14 | **Distributed message queue** | sharding, replication, ordering, visibility timeout, exactly-once/idempotency (ties Cluster 11) |
| 15 | **Semantic search / RAG** | vector DB (embeddings, ANN), chunking, retrieval + ranking (ties Cluster 6) |

**Cross-cutting deep-dive building blocks reused across designs:** consistent hashing, Bloom filter / count-min sketch, idempotency keys, CDC / dual-write, read-replica vs cache for read scaling, hot-key/hotspot mitigation, presigned-URL upload, WebSocket real-time, distributed locks/fencing.

---

## 5. Best Sources (deduplicated)

| Source | URL | Best for |
|---|---|---|
| **donnemartin/system-design-primer** (GitHub) | https://github.com/donnemartin/system-design-primer | Canonical free breadth/glossary: scalability, LB, cache, DNS/CDN, RDBMS, NoSQL, sharding, consistent hashing, reverse proxy; worked URL-shortener/Twitter/crawler |
| **Hello Interview — System Design in a Hurry** (+ Deep Dives, Problem Breakdowns) | https://www.hellointerview.com/learn/system-design | Most current (2024-26) interview framing: delivery framework, caching, sharding, consistent hashing, CAP, API design, Kafka, Cassandra, vector DBs, rate limiter; worked Ticketmaster/Dropbox/Top-K |
| **ByteByteGo / Alex Xu — System Design Interview Vol 1 & 2** | https://bytebytego.com/ | 4-step framework + canonical worked designs (rate limiter, Snowflake, URL shortener, crawler, notifications, newsfeed, chat, autocomplete, YouTube, Drive); back-of-envelope; B-Tree vs LSM; consistent hashing; CDN |
| **Designing Data-Intensive Applications — Kleppmann (DDIA)** | https://dataintensive.net/ | Deep data-layer reference (Days 5-11, 17-19): transactions/isolation, replication + lag anomalies, partitioning/rebalancing, consensus, quorums, fencing tokens, logical clocks, CDC, storage engines. Does NOT cover proxies/mesh |
| **DesignGurus — Grokking the System Design Interview / blog** | https://www.designgurus.io/ | Building-blocks→case-study structure; scalability, CAP vs PACELC, caching, replication, APIs, security decisions, 2026 roadmap |
| **Google SRE Book** | https://sre.google/sre-book/monitoring-distributed-systems/ | Authoritative Golden Signals, SLI/SLO/SLA, error budgets; rule-of-thumb latency PDF |
| **Jeff Dean — Latency Numbers gist** | https://gist.github.com/jboner/2841832 | The canonical latency table for Day 15 |
| **Cloudflare Learning Center (DNS/Anycast/records)** | https://www.cloudflare.com/learning/dns/what-is-dns/ | Primary-source DNS chain, recursive vs authoritative, record types, Anycast, TTL |
| **AWS docs — ELB (ALB/NLB, draining) + Route 53 routing policies** | https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html | Cloud LB product mapping, connection draining, GSLB/DNS routing taxonomy |
| **Confluent — Exactly-Once Semantics + Consumer Design docs** | https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/ | Kafka delivery semantics, idempotent producers, consumer groups/offsets/rebalancing |
| **microservices.io (Chris Richardson) — Saga, Outbox, Idempotent Consumer** | https://microservices.io/patterns/data/saga.html | Canonical cross-service consistency patterns |
| **Martin Kleppmann — How to do distributed locking** | https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html | Redlock critique + why fencing tokens are required |
| **raft.github.io** | https://raft.github.io/ | Primary Raft consensus reference + visualization |
| **Stripe — Designing APIs with idempotency** | https://stripe.com/blog/idempotency | Idempotency-key design for payments |
| **Vlad Mihalcea — Optimistic vs Pessimistic Locking** | https://vladmihalcea.com/optimistic-vs-pessimistic-locking/ | Concrete SQL locking strategies (Day 5) |
| **ByteByteGo blog — B-Trees vs LSM-Trees** | https://blog.bytebytego.com/p/b-trees-vs-lsm-trees-comparison-and | SSTables, compaction, Bloom filters, write/read amplification |
| **System Design School — spatial indexes (QuadTree/Geohash/H3/S2)** | https://systemdesignschool.io/domain-knowledge/choosing-spatial-index | Geospatial indexing for Uber/Yelp designs |
| **Microsoft Azure Architecture Center — Cloud Design Patterns + CI/CD** | https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker | Circuit breaker/bulkhead/retry definitions; blue-green/canary/rolling; mTLS/zero-trust |
| **AWS Architecture Blog — Exponential Backoff and Jitter** | https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/ | Definitive retry-jitter / thundering-herd reference |
| **Alex DeBrie — DynamoDB Single-Table Design** | https://www.alexdebrie.com/posts/dynamodb-single-table/ | Access-pattern-driven KV/document key modeling |
| **NGINX docs — HTTP LB + HA with keepalived** | https://nginx.org/en/docs/http/load_balancing.html | Nginx as reverse proxy/L7 LB; VIP/VRRP/keepalived HA ("who balances the balancer") |
| **High Scalability — Gossip Protocol Explained** | https://highscalability.com/gossip-protocol-explained/ | Gossip/anti-entropy + heartbeat membership (Cassandra/Dynamo/Consul) |

Source emphasis map (meta): **primer** = breadth; **Alex Xu** = framework + worked designs; **DDIA** = data-layer depth (no proxies/mesh); **Hello Interview** = current interview framing + problem breakdowns; **Grokking** = structured building-blocks→cases.