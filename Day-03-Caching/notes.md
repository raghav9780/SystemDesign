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
- **RAM read: ~100 ns** — where a cache lives.
- **SSD read: ~100 µs** — about **1,000× slower** than RAM.
- **DB query over the network: ~1–10 ms** — about **10,000–100,000× slower** than RAM.

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
| Data structures | Rich (lists / sets / sorted sets / hashes) | Strings only |
| Persistence | Yes (can save to disk) | No (pure in-memory) |
| Replication / HA | Yes | No |
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

**Distributed caching + consistent hashing tie-in (cross-links Day 9).** When your cache is a *cluster*, how do you decide which node holds a key? The naive answer, `hash(key) % N`, is a trap: change `N` (add or lose one node) and **almost every key remaps to a different node** — that's a self-inflicted **avalanche**, the entire cache effectively cold at once. **Consistent hashing** (keys and nodes placed on a **ring**, with **virtual nodes** for even spread) moves only **~1/N of keys** on a membership change instead of nearly all of them. This is exactly why **Redis Cluster** uses fixed **hash slots** — so scaling the cluster doesn't nuke the cache.

**Cache hit ratio — the effectiveness metric.** `hit ratio = hits / (hits + misses)`. It's the single number that tells you whether the cache is earning its keep, and it drives your **capacity math**: at a **90% hit ratio**, an incoming **100k read QPS** turns into only **~10k QPS hitting the DB** (the other 90% is served from RAM). That 10× reduction is the whole argument for the cache in a sizing discussion — quote the ratio, then do the multiply.

**When NOT to cache + cache warming.** Caching isn't free, and some data shouldn't be cached at all: **write-heavy, low-reuse data** (you pay to cache it and nobody reads it back) and **data that needs strong consistency** — money, balances, inventory you can't oversell — where a stale read is a correctness bug, not a minor annoyance. Separately, **cache warming** is the practice of **pre-loading known-hot keys at startup** (from a script or a warm-up job) so that right after a deploy or restart you don't serve everyone off a **cold cache** — which would mean a flood of misses and a nasty latency spike exactly when the box is freshest.

**Bloom filter for penetration (alongside null-caching).** Null-caching (storing a "not found" marker) fixes penetration *after* the first miss; a **Bloom filter** rejects the bogus lookup *before* it ever touches the cache or DB. A Bloom filter is a tiny probabilistic structure that answers **"this key definitely does NOT exist"** (or "might exist") in **O(1) time and very little memory**. So for a stream of malicious/nonexistent IDs, you check the filter first: a "definitely not there" answer is rejected instantly, and only *possible* keys are allowed through to the real lookups. (It can have false positives — "might exist" when it doesn't — but never false negatives, which is exactly the guarantee you need here.)
