# Day 3 — Caching
**Phase 1: Foundations** | Score: 6.5 → 8

> **One-sentence idea:** A cache is a small, FAST copy of your hottest data kept close by,
> so you stop paying the slow cost of fetching it from the real source every time. Think of a
> **chef keeping the 10 most-ordered ingredients on the counter** instead of walking to the
> freezer for each dish — the whole craft is deciding *what to keep on the counter, when it
> goes stale, and what happens when everyone orders the same thing at once.*

## Core Idea — why a cache works at all
A cache is a small, FAST, in-memory store that holds your **hot data** (the data that gets
asked for most), so you can avoid hitting the slow source of truth — your database, disk, or
an external API — on every request.

Why does such a tiny store help so much? Because real traffic is **skewed**, not uniform: as a
rule of thumb, **~90% of traffic wants ~10% of the content.** Most people load the same few viral
tweets, the same homepage, the same top products. So you only need to keep that hot 10% nearby to
absorb the bulk of the load. ⭐ This skew is the entire reason caching is worth it — if every
request wanted a different random row, a small cache would almost never help.

## Speed gut-feel (memorize these)
The reason we go to all this trouble is the raw gap in access speed between memory and everything
else. Burn these into memory — they justify every caching decision:
- **RAM reference: ~100 ns** — where a cache lives.
- **SSD *random* read (one small 4 KB block): ~100–150 µs** (the classic Jeff Dean table says 150 µs) —
  about **1,000× slower** than a RAM reference.
- **DB query over the network: ~1–10 ms** — about **10,000–100,000× slower** than RAM.

⚠️ **Precision — the "1,000×" is for RANDOM access only.** If instead you read **1 MB *sequentially***, RAM
takes **~250 µs** and an SSD **~1 ms** — only about **4× apart**. Why the gap shrinks: a *random* read pays the
device's fixed **per-operation latency** every single time (find the block, fetch it, hand it back), and for a
tiny read that fixed cost is almost the whole bill. A *sequential* read pays that setup cost once and then
**streams** bytes at the device's throughput, where SSDs are much closer to RAM. Caches mostly serve lots of
small, scattered lookups ("user 42", "post 9001") — i.e. random access — which is exactly where RAM's ~1,000×
edge is real. (Same framing as Day 15's latency table.)

So serving a value from RAM instead of a network DB query isn't a small win; it's the difference
between sub-microsecond and multi-millisecond. That multiplier is why caching is one of the first
levers you reach for when something is slow.

## Cache layers — caching happens at many levels
"The cache" isn't one box; a request can be served from several caches on its way in. From closest
to the user to furthest:
- **Browser (client) cache** — the user's own device remembers a response.
- **CDN (edge) cache** — servers geographically near the user hold copies (great for images, static files).
- **Application cache (Redis / Memcached)** — the shared in-memory store your backend checks. (A
  **CDN**, Content Delivery Network, is just a network of edge servers near users; **Redis/Memcached**
  are the standalone in-memory stores covered below.)
- **DB internal cache** — the database keeps recently-used pages in its own memory too.

The point: a value you "cached" might be served before the request ever reaches your code. Each layer
shaves latency off a different leg of the trip.

---

## 3 Caching Strategies ⭐ (drilled in interviews)
The three strategies differ in **one question: when do the cache and the database get written, relative
to each other?** Get that framing and the trade-offs fall out naturally.

### 1. Cache-Aside (Lazy Loading) — MOST COMMON
Here the **application** manages the cache directly, and data is loaded **lazily** — only when someone
actually asks for it. The flow: the app checks the cache first; on a **hit** (the value is there) it
returns immediately; on a **miss** (not there) it reads the DB, **writes that value into the cache**,
then returns it. So the cache fills up on demand, one requested key at a time.

Think of it as a librarian who only fetches a book from the back archive when a reader requests it —
and then leaves it on the front desk for the next person.

- ✅ **Only requested data gets cached** (you never waste memory on data nobody reads), and if the
  **cache goes down the app still works** — it just falls back to reading the DB (slower, but alive).
- ❌ The **first request for any key is always a miss** (the "cold start" penalty), and the cached copy
  **can go stale** if the DB changes underneath it.

### 2. Write-Through
Every write goes to **both the cache AND the database, synchronously, together** — you don't return
"done" until both are updated. The cache is kept in lock-step with the source of truth on every write.

Like writing an appointment into both your phone and your paper planner at the same moment before you
move on — they can never disagree.

- ✅ The cache is **never stale** — it always matches the DB.
- ❌ Writes are **slower** (you pay for two writes every time), and you end up **caching data that may
  never be read** (you wrote it to cache "just in case," wasting memory on cold items).

### 3. Write-Back (Write-Behind)
Write to the **cache only and return immediately**; the cache **flushes to the DB asynchronously later**
(in a batch, or after a short delay). The user's write feels instant because it only touched fast memory.

Like jotting orders on a sticky note and only entering them into the official system every few minutes —
fast in the moment, but the sticky note is the only record until then.

- ✅ **Blazing-fast writes**, and it **absorbs write spikes** beautifully — a sudden flood of writes hits
  fast RAM, and the slow DB drains them gradually.
- ❌ ⭐ **The risk: if the cache dies before it flushes, that data is LOST forever** — it was never written
  to the durable DB. So use this **only where some loss is acceptable**: view counts, metrics, like counts.
  **ALWAYS state this data-loss risk out loud** when you propose write-back — naming the risk is the point.

---

## Eviction Policies — the cache is full, what gets kicked out?
A cache is deliberately small, so eventually it fills up and you must evict something to make room. The
**eviction policy** is the rule for choosing the victim. Pick wrong and you evict the item everyone's
about to ask for.

- **LRU (Least Recently Used)** — evict whatever hasn't been *touched* for the longest time. The bet: if
  you haven't needed it lately, you probably won't need it soon. ⭐ This is the sensible **default**.
- **LFU (Least Frequently Used)** — evict whatever has been *accessed the fewest times* overall. Favors
  long-term popularity over recency.
- **FIFO (First In, First Out)** — evict the **oldest inserted** item, regardless of how often it's used.
  Simple, but can dump a hot item just because it arrived early.
- **TTL (Time To Live)** — each entry **auto-expires after N seconds**. Not really "what to kick out when
  full" so much as "everything is allowed to get stale only so long"; commonly **combined with LRU** (LRU
  decides what to drop when full, TTL guarantees nothing lives past its freshness window).

---

## Famous Cache Problems ⚠️ (NAME THEM)
These are the classic failure modes. In an interview, the key skill is to **say the name** of the problem,
then give the fix — naming it proves you recognize the pattern.

**Stale data** — the DB got updated but the cache still holds the old value, so readers see outdated
information. **Fix:** set a **TTL** so the stale copy can't live long, or **invalidate on write** (drop/refresh
the cached key the moment the underlying data changes).

**Cache Stampede / Thundering Herd** — a single **hot key expires**, and in the instant it's gone,
*thousands of simultaneous requests* all miss the cache and **slam the database at once**, each trying to
rebuild the same value. The DB can buckle under that synchronized surge. Three standard fixes:
- **Single-flight / lock** — only **one** request is allowed to rebuild the value; all the others **wait**
  for it and then reuse the result. One DB hit instead of thousands.
- **Jittered / staggered TTLs** — add a little randomness to expiry times so a batch of keys **don't all
  expire at the same instant** and trigger a coordinated stampede.
- **Refresh-ahead** — proactively **refresh hot keys *before* they expire**, so the value is always warm and
  there's never a gap where everyone misses at once.

**Cache Penetration** — requests for **keys that don't exist** (e.g. a bogus or malicious ID) always miss
the cache by definition, so they **bypass it entirely and hammer the DB** every single time. **Fix:** **cache
the "not found" result too** — store a marker that says "this key has no value," so repeat lookups for the
missing key are answered by the cache instead of the DB.

---

## Redis vs Memcached
Both are in-memory application caches; the difference is how much each can do. Redis is the richer, more
capable system; Memcached is the leaner, simpler one.

| | Redis | Memcached |
|---|---|---|
| Data structures | Rich (lists / sets / sorted sets / hashes) | Plain key → opaque value (string/bytes) only |
| Persistence | Yes (can save to disk — RDB / AOF, see pass-2 section) | No (pure in-memory) |
| Replication / HA | Yes (replicas + Sentinel, or Redis Cluster) | No built-in replication |
| Threading | Mostly single-threaded | Multi-threaded |

- **DEFAULT: Redis.** Reach for it when you want **rich data structures** (e.g. a **sorted set** is perfect
  for rankings/leaderboards because it keeps items ordered by score), **persistence** (data survives a
  restart), or **HA** (High Availability — replicas keep the cache alive if one node dies).
- ❌ ⭐ **Do NOT justify Redis by saying it's "single-threaded."** Being mostly single-threaded is a
  **limitation**, not a selling point — Memcached is actually multi-threaded. Justify Redis with the *real*
  reasons above (structures, persistence, HA), not with its threading model.

---

## Extra Lessons (from follow-up + retry)
A few sharper distinctions that came out of the follow-up questions and the retry — these are exactly where
the "match the tool to the problem" judgment lives:

- **Durability beats speed for money/legal data.** If data must not be lost (payments, legal records),
  write-back is the wrong choice because of its loss risk. Either switch to **write-through** (synchronous to
  the durable DB, never stale), or **buffer writes in a durable queue like Kafka** — which gives you *both*
  burst absorption *and* durability (a **durable queue** persists the messages, so a crash doesn't lose them).

- **Invalidate-on-write vs write-through — don't blur them.** **Invalidate-on-write** means you **DELETE the
  cache key** when the underlying data is edited, so the *next read* misses and repopulates it fresh.
  **Write-through** means you **UPDATE both the cache and the DB** at write time. One removes the stale entry
  and lets it be rebuilt lazily; the other proactively rewrites it. Different mechanisms — keep the names straight.

- **Expensive-to-COMPUTE reads → precompute + cache.** When the cost isn't fetching the data but *computing*
  it (e.g. a "Trending" list that requires crunching lots of rows), don't compute it on the **hot path** (during
  the user's request). Instead **precompute it with a background job and cache the result with a TTL**, so users
  just read the ready-made answer. ⚠️ This is **NOT** write-back — write-back is about absorbing *writes*, not
  about avoiding expensive *reads*.

- ⭐ **The principle that ties it together:** **write-back absorbs bursty WRITES; precompute+cache serves
  expensive COMPUTES.** Match the strategy to the *shape* of the problem, not just its name.

---

## Personal Notes (recurring gaps — improving!)
- [x] NAME the concept (retry: named "cache stampede", gave single-flight + TTL fixes ✅).
- [x] FINISH the thought (retry: stated trade-offs throughout ✅).
- [ ] Justify with REAL reason (Redis = structures/persistence/HA, NOT single-threaded).
- [ ] Match strategy to PROBLEM SHAPE (don't use write-back for computed values).

---

## ⭐ Interview-completeness additions (audit pass)

**Cache Avalanche — the missing 3rd failure mode.** The classic trio is stampede / penetration / **avalanche**, and this one is the "big" version. Where a *stampede* is **one hot key** expiring, an *avalanche* is a **large SET of keys expiring at the same instant** (or a whole cache node dying) — so a huge fraction of your traffic misses *all at once* and the DB gets swamped broadly, not just on one row. Picture every ingredient on the chef's counter spoiling at midnight together: suddenly every single order means a trip to the freezer. **Fixes:** **TTL jitter** (randomize expiries so keys don't all die on the same tick), **multi-level caching** (an L1 layer absorbs the gap while the main cache refills), and **circuit breakers / throttling** (shed or slow load so the DB survives the surge). Mnemonic: *stampede = one hot key; avalanche = many keys / whole node.*

**Read-through and write-around — the other 2 of the 5 patterns.** The notes above teach cache-aside, write-through, and write-back; two more round out the set. **Read-through:** instead of the app checking the cache and fetching the DB on a miss (that's cache-aside), the **cache library/service itself owns the DB fetch** — the app *only ever talks to the cache*, and the cache transparently loads from the DB behind the scenes. Cleaner app code; the caching layer becomes the single data access point. **Write-around:** writes go **straight to the DB, bypassing the cache entirely**; the cache only fills later when someone actually reads that key (a normal cache-aside miss). The win: you avoid **flooding the cache with write-heavy data that's rarely read back** — no point warming the counter with dishes nobody orders.

**Hot key (celebrity) problem.** This is *not* a miss problem — the key is happily cached and never expires. The trouble is **access skew**: one key (a celebrity's profile, a viral post) gets so much traffic that **the single cache node holding it becomes the bottleneck**, maxing out that one node's CPU/network while the rest of the cluster sits idle. **Fixes:** **replicate or split the key across nodes** (store it as `key#1 … key#N` and read a random replica, spreading the load), add a **local in-process L1 cache** on each app server (so the hot value is answered before it even reaches Redis), or give it a **dedicated node**.

**The dual-write / invalidation-ordering race.** When a write must touch *both* the DB and the cache, **the order matters** — get it wrong and you cache stale data *permanently*. If you **invalidate (delete) the cache first, then write the DB**, a concurrent reader can slip in *after* the delete but *before* your DB write lands: it reads the OLD value from the DB and **repopulates the cache with the stale value** — now the cache is wrong forever. **Safer order: write the DB first, then DELETE the key** (delete, don't *update* — deleting sidesteps a similar race between two concurrent writers' updates). And always keep a **TTL as the backstop** so even a lost race self-heals eventually.

**Distributed caching + consistent hashing tie-in (cross-links Day 9).** When your cache is a *cluster*, how do you decide which node holds a key? The naive answer, `hash(key) % N`, is a trap: change `N` (add or lose one node) and **almost every key remaps to a different node** — that's a self-inflicted **avalanche**, the entire cache effectively cold at once. **Consistent hashing** (keys and nodes placed on a **ring**, with **virtual nodes** for even spread) moves only **~1/N of keys** on a membership change instead of nearly all of them. **Redis Cluster** solves the same problem with a *different* mechanism — fixed **hash slots** (not a ring; see "Redis HA" in the pass-2 section below) — so scaling the cluster moves only some slots and doesn't nuke the cache.

**Cache hit ratio — the effectiveness metric.** `hit ratio = hits / (hits + misses)`. It's the single number that tells you whether the cache is earning its keep, and it drives your **capacity math**: at a **90% hit ratio**, an incoming **100k read QPS** turns into only **~10k QPS hitting the DB** (the other 90% is served from RAM). That 10× reduction is the whole argument for the cache in a sizing discussion — quote the ratio, then do the multiply.

**When NOT to cache + cache warming.** Caching isn't free, and some data shouldn't be cached at all: **write-heavy, low-reuse data** (you pay to cache it and nobody reads it back) and **data that needs strong consistency** — money, balances, inventory you can't oversell — where a stale read is a correctness bug, not a minor annoyance. Separately, **cache warming** is the practice of **pre-loading known-hot keys at startup** (from a script or a warm-up job) so that right after a deploy or restart you don't serve everyone off a **cold cache** — which would mean a flood of misses and a nasty latency spike exactly when the box is freshest.

**Bloom filter for penetration (alongside null-caching).** Null-caching (storing a "not found" marker) fixes penetration *after* the first miss; a **Bloom filter** rejects the bogus lookup *before* it ever touches the cache or DB. A Bloom filter is a tiny probabilistic structure that answers **"this key definitely does NOT exist"** (or "might exist") in **O(1) time and very little memory**. So for a stream of malicious/nonexistent IDs, you check the filter first: a "definitely not there" answer is rejected instantly, and only *possible* keys are allowed through to the real lookups. (It can have false positives — "might exist" when it doesn't — but never false negatives, which is exactly the guarantee you need here.)

---

## ⭐ Completeness additions (pass 2)

These close the remaining gaps against the Day 3 checklist. Same voice as above: the idea, a small example,
why it matters, then the line to say in an interview.

### 1. Cache-aside in actual code (with TTL, jitter, null-caching, delete-on-write)
Everything above is easier to remember once you've seen it as ~20 lines. Python-style pseudocode with Redis:

```python
import json, random

BASE_TTL = 300                       # 5 minutes
NULL_MARKER = "__NULL__"             # our "this key does not exist" sentinel

def get_user(user_id):
    key = f"user:{user_id}"
    cached = redis.get(key)                      # 1. check the cache first
    if cached is not None:                       # 2. HIT
        return None if cached == NULL_MARKER else json.loads(cached)

    user = db.query("SELECT * FROM users WHERE id = %s", user_id)   # 3. MISS -> go to DB
    if user is None:
        redis.set(key, NULL_MARKER, ex=60)       # 4a. penetration fix: cache "not found" briefly
        return None
    ttl = BASE_TTL + random.randint(0, 60)       # 4b. TTL + jitter (avalanche fix)
    redis.set(key, json.dumps(user), ex=ttl)     #     store it for the next reader
    return user

def update_user(user_id, fields):
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)   # 1. write the DB FIRST
    redis.delete(f"user:{user_id}")                             # 2. then DELETE (not update) the key
```

Read it line by line: the read path is the cache-aside flow from the top of the notes; `ex=` is the TTL in
seconds (Redis expires the key for you); the `random.randint` is **TTL jitter**; the `NULL_MARKER` is
**null/negative caching** with a deliberately *short* TTL (so if the user is created later, the "not found"
doesn't linger); and `update_user` uses the **safe order — DB first, then delete** — from the dual-write race
section. If an interviewer says "write the cache-aside code," this is the answer.

### 2. The two missing eviction policies — MRU and Random
The earlier list had LRU, LFU, FIFO (+TTL). Two more complete the set:
- **MRU (Most Recently Used)** — evict the item you touched **most recently**. Sounds backwards, so think of
  when it's right: a **repeated sequential scan** bigger than the cache (e.g. a job reading rows 1 → 1,000,000
  over and over). Under LRU, each new row evicts the oldest one — exactly the one the scan will need next lap —
  so the hit ratio collapses to ~0. MRU keeps the *older* items and throws away the one you just finished with,
  which you won't need again until the next lap. Niche, but it's the textbook answer to "when is LRU wrong?"
- **Random** — evict a random key. No bookkeeping at all (no recency list, no counters), so it's the cheapest
  policy, and when access is fairly uniform it performs close to LRU. Use it when the memory/CPU cost of
  tracking matters more than squeezing out the last few % of hit ratio.

**How Redis really does it:** Redis has a `maxmemory-policy` setting. The open-source default is
**`noeviction`** — when memory is full, writes get an **error** instead of evicting anything (fine for a
database, wrong for a cache!). For a cache you set e.g. **`allkeys-lru`** or **`allkeys-lfu`**; other options
include `allkeys-random` and `volatile-*` variants (`volatile-lru`, `volatile-lfu`, `volatile-random`,
`volatile-ttl`) that only evict keys that *have* a TTL set. Redis's LRU/LFU are **approximated**: it samples a
handful of keys and evicts the best candidate among them rather than keeping a perfect global ordering —
much cheaper, nearly as good. Interview line: *"I'd set maxmemory with allkeys-lru (or allkeys-lfu for stable
popularity) — the default noeviction would make writes fail once the cache fills."*

### 3. The full stampede toolkit (name all of them)
Earlier: single-flight/lock, jittered TTLs, refresh-ahead. The complete menu an interviewer may probe:
- **Per-key mutex / distributed lock.** On a miss, try to grab a lock for that key; only the winner rebuilds.
  In Redis: `SET lock:user:42 <token> NX PX 5000` — `NX` = only set if nobody holds it, `PX 5000` = auto-expire
  in 5 s so a crashed rebuilder can't block everyone forever. Losers wait briefly and re-read the cache (or
  serve a stale copy). (Safe lock release with a token, and why distributed locks are subtle, is Day 18.)
- **Request coalescing (a.k.a. single-flight).** The *in-process* version: if 500 requests on the **same app
  server** miss the same key at once, that server makes **one** DB call and hands the one result to all 500
  waiters. Cheap, no network lock — but each app server still makes its own one call, so it pairs well with
  the distributed lock above.
- **Stale-while-revalidate.** Keep two clocks per entry: a "fresh until" time and a later "hard expiry." Between
  them, serve the **stale value immediately** and kick off **one background refresh**. Users never wait on the
  rebuild and the DB sees one query. (Same idea as the HTTP `stale-while-revalidate` directive on Day 4.)
- **Probabilistic early expiration (XFetch).** Instead of everyone noticing expiry at the same instant, each
  reader rolls a die: the **closer the key is to expiring** (and the **longer it takes to recompute**), the
  likelier that *this* reader decides to refresh it **early**. One lucky reader refreshes a bit before the
  deadline, so under load the key never actually goes cold. The published rule (Vattani et al., VLDB 2015):
  refresh if `now − δ·β·ln(rand()) ≥ expiry`, where δ = how long a recompute takes and β ≈ 1 (`ln(rand())` is
  negative, so this adds a random "look-ahead"). No lock needed.
- **Refresh-ahead** (already above) — a background job refreshes known-hot keys before they expire.

Interview line: *"For a hot key I'd use a per-key lock or request coalescing so only one rebuild hits the DB,
plus stale-while-revalidate or probabilistic early expiry so the key effectively never goes cold."*

### 4. Redis persistence — RDB vs AOF (and why a *cache* cares)
A pure cache could lose everything on restart — but then you restart into a **cold cache** and stampede your
DB. And many teams also keep sessions/leaderboards in Redis, where loss really hurts. Redis has two ways to put
memory onto disk:
- **RDB (snapshots).** Every so often (e.g. "every 60 s if ≥1,000 keys changed"), Redis **forks** a child
  process that writes a compact point-in-time copy of the whole dataset to one file (`dump.rdb`). Like
  **photographing the whiteboard** every few minutes. ✅ Compact, great for backups, fast restarts. ❌ Crash
  between snapshots → you lose **everything since the last photo** (often minutes). The fork can also briefly
  stall a very large instance.
- **AOF (Append-Only File).** Every write command (`SET`, `INCR`…) is **appended to a log**; on restart Redis
  **replays the log**. Like **writing every change into a diary**. How often it's forced to disk is
  `appendfsync`: `always` (safest, slowest), **`everysec` (the default — lose at most ~1 s of writes)**, or
  `no` (let the OS decide). ✅ Far more durable. ❌ Bigger files, slower restarts; Redis periodically
  **rewrites/compacts** the log in the background.
- **Both together** is Redis's own recommendation when you care about the data — AOF for durability, RDB for
  backups/fast restarts. If both exist, Redis restores from the AOF (the more complete one).
- Default config: RDB snapshots on; AOF off until you set `appendonly yes`.

Interview line: *"RDB = periodic snapshot, compact but can lose minutes; AOF = write log, loses ≤1 s with
everysec. For a pure cache I might accept RDB or nothing; for sessions I'd turn on AOF."*

### 5. Redis high availability — Sentinel vs Redis Cluster
Day 2's SPOF ladder said "Redis is now a SPOF → add replication + failover." Here's the actual machinery.
- **Replication (the base layer).** One **primary** takes writes; **replicas** copy it (asynchronously) and can
  serve reads — the single-leader idea from Day 8. But who promotes a replica when the primary dies?
- **Redis Sentinel = automatic failover for ONE primary.** A small group of **Sentinel** processes (run ≥3 so
  they can out-vote each other) watch the primary. When a **quorum** of them agree it's down, they **promote a
  replica** and tell clients the new address (clients ask Sentinel "who's the primary?"). Sentinel gives you
  **HA but not more capacity** — all data must still fit on, and all writes still go to, one primary.
- **Redis Cluster = sharding + failover built in.** The keyspace is split into **16,384 hash slots**: each key
  goes to slot `CRC16(key) mod 16384`, and each primary owns a range of slots (3 primaries ≈ 5,461 slots each).
  Each primary has replicas, and the cluster itself promotes one if a primary dies — **no Sentinel needed**.
  Adding a node = **moving some slots** to it, so only those keys relocate (the same "don't remap everything"
  goal as consistent hashing, achieved with a fixed slot table instead of a ring). Catch: a command touching
  **several keys needs them in the same slot** — you force that with **hash tags**: `{user42}:cart` and
  `{user42}:profile` hash only the `user42` part, so they land together.
- ⚠️ In both, replication is **asynchronous**, so a failover can **lose the last few acknowledged writes**
  (Day 8's async gap). Fine for a cache; know it for anything else.

Rule of thumb: **data fits on one node and just needs to survive → Sentinel. Data or write load too big for one
node → Cluster.**

### 6. How does a key find its node? Client-side hashing vs Redis Cluster
Two ways to organise a distributed cache:
- **Client-side sharding** (classic **Memcached**): the cache servers don't know about each other. The
  **client library** hashes each key with **consistent hashing** (ring + virtual nodes, Day 9) to pick a
  server. Simple servers, but every client must share the same server list and hashing scheme.
- **Server-side slots** (**Redis Cluster**): the cluster owns the slot map. A cluster-aware client caches the
  map and goes straight to the right node; if the slot has moved, the node replies **`MOVED`** with the correct
  address and the client updates its map.

### 7. Redis vs Memcached — the complete decision criteria
On top of the table above: Redis also has **pub/sub** (and streams), **Lua scripting / atomic multi-step
operations**, and **configurable eviction policies**; Memcached's selling points are **simplicity** and being
**multi-threaded**, so one big many-core box scales well for plain `get/set` of blobs. Decision: **need data
structures, persistence, replication/failover, or pub/sub → Redis (the usual default). Need only a simple,
large, multi-threaded key→blob cache → Memcached is a perfectly good choice.**

### 8. Multi-level caching — the full layer map, with L1 vs L2
The layer list near the top skipped one important layer. Nearest to farthest:
**browser → CDN → app-server L1 (in-process memory) → distributed L2 (Redis/Memcached) → DB buffer pool**.
- **L1 = in-process** — an LRU map inside each app server's own memory (e.g. Caffeine in Java). No network hop
  at all: the fastest cache you can have, and the classic hot-key fix.
- **L2 = shared distributed cache** (Redis) — one network hop (sub-millisecond inside a data center), but *one
  shared copy* every server sees.
- **The catch with L1:** there are now **N separate copies** (one per server), each going stale on its own —
  deleting the Redis key does not clear 50 servers' L1s. So L1 gets **very short TTLs (seconds)**, or you
  broadcast invalidations (e.g. over Redis pub/sub). Use L1 only for data that tolerates a few seconds of staleness.

### 9. Bloom filter — two caveats interviewers check
- It only works if you **load every valid key into it** (e.g. all existing user IDs) and **add each new key on
  creation**. A real key missing from the filter would be wrongly rejected — so keep it in sync.
- A standard Bloom filter **can't delete** items (clearing bits could erase other keys' bits). Deleted keys just
  become harmless false positives; if you need deletion, use a counting Bloom filter or rebuild periodically.

### 10. Avalanche when a cache NODE dies (not just mass expiry)
The avalanche fixes above focus on synchronized expiry. If a whole cache node dies, the fix is **cache HA** —
replicas + automatic failover (Sentinel/Cluster, section 5) — plus **rate limiting / circuit breakers** in front
of the DB (Day 14, Day 21) so the DB sheds load instead of collapsing while the cache refills.

### 11. Quick reference (summary — the teaching is above)
| Problem | What it is | Fixes |
|---|---|---|
| Stampede | ONE hot key expires, many rebuild at once | per-key lock, request coalescing, stale-while-revalidate, XFetch, refresh-ahead |
| Penetration | keys that exist nowhere bypass the cache | null caching (short TTL), Bloom filter, input validation |
| Avalanche | MANY keys expire together / a node dies | TTL jitter, multi-level cache, HA replicas, circuit breaker/throttling |
| Hot key | one key overloads one node (no miss involved) | L1 cache, key splitting `key#1..N`, dedicated node |

### 12. Putting it together — "Black Friday is coming, how do you cache?"
A skeleton answer that uses everything: **warm** the cache with known-hot products before the sale; **L1 + key
splitting** for the few celebrity products; **TTL jitter** so the warmed keys don't all expire together;
**stale-while-revalidate / per-key locks** against stampedes; **don't treat cached inventory as truth**
(strong-consistency data — check stock in the DB at checkout); an **HA cache** (Cluster with replicas); and
**rate limiting + circuit breakers** in front of the DB as the last line of defense.
