# Day 3 — Caching
**Phase 1: Foundations** | Score: 6.5/10

---

## Core Idea
- Cache = small FAST in-memory store holding hot data, so you avoid the slow source (DB/disk/API).
- Works because access is SKEWED: ~90% of traffic wants ~10% of content.

## Speed gut-feel (memorize)
- RAM read: ~100 ns
- SSD read: ~100 µs (1,000x slower)
- DB query over network: ~1–10 ms (10,000–100,000x slower)

## Cache layers
- Browser (client) → CDN (edge) → Application cache (Redis/Memcached) → DB internal cache.

---

## 3 Caching Strategies ⭐ (drilled in interviews)

### 1. Cache-Aside (Lazy Loading) — MOST COMMON
- App checks cache. Hit → return. Miss → read DB, write cache, return.
- ✅ Only requested data cached; cache down ≠ app down.
- ❌ First request per key = miss; data can go stale.

### 2. Write-Through
- Write to cache AND DB synchronously together.
- ✅ Never stale.  ❌ Slower writes; caches data that may never be read.

### 3. Write-Back (Write-Behind)
- Write to cache only, return now; flush to DB async later.
- ✅ Blazing writes; absorbs write spikes.
- ❌ RISK: cache dies before flush → DATA LOST. Use only where some loss is OK
  (view counts, metrics, like counts). ALWAYS state this risk out loud.

---

## Eviction Policies (cache full — what to kick out?)
- **LRU** (Least Recently Used) — evict longest-untouched. ⭐ default.
- **LFU** (Least Frequently Used) — evict fewest-accessed.
- **FIFO** — evict oldest inserted.
- **TTL** — auto-expire after N sec (combine with LRU).

---

## Famous Cache Problems ⚠️ (NAME THEM)
- **Stale data** — DB updated, cache old. Fix: TTL or invalidate-on-write.
- **Cache Stampede / Thundering Herd** — hot key expires → thousands of simultaneous
  misses slam DB at once. Fixes:
    - **Single-flight / lock** — only 1 request rebuilds, others wait.
    - **Jittered/staggered TTLs** — don't expire everything at the same instant.
    - **Refresh-ahead** — refresh hot keys BEFORE they expire.
- **Cache Penetration** — requests for non-existent keys bypass cache, hammer DB.
  Fix: cache the "not found" result too.

---

## Redis vs Memcached
| | Redis | Memcached |
|---|---|---|
| Data structures | Rich (lists/sets/sorted sets/hashes) | Strings only |
| Persistence | Yes | No |
| Replication/HA | Yes | No |
| Threading | Mostly single-threaded | Multi-threaded |
- DEFAULT: **Redis** — pick it for rich structures (sorted sets = ranking), persistence, HA.
- ❌ DO NOT justify Redis with "single-threaded" — that's a LIMITATION, not a feature.

---

## Extra Lessons (from follow-up + retry)
- Money/legal data → durability beats speed. Write-back (loss risk) → switch to
  **write-through**, or buffer in a **durable queue (Kafka)** for burst + durability.
- **Invalidate-on-write** = DELETE the cache key on edit (next read repopulates).
  Different from **write-through** = UPDATE both cache+DB. Don't blur the two.
- Expensive-to-COMPUTE read data (e.g. "Trending" list) → **precompute via background
  job + cache with TTL**, never compute on the hot path. (NOT write-back — that's for writes.)
- **Principle:** write-back absorbs bursty WRITES; precompute+cache serves expensive COMPUTES.

## Personal Notes (recurring gaps — improving!)
- [x] NAME the concept (retry: named "cache stampede", gave single-flight + TTL fixes ✅).
- [x] FINISH the thought (retry: stated trade-offs throughout ✅).
- [ ] Justify with REAL reason (Redis = structures/persistence/HA, NOT single-threaded).
- [ ] Match strategy to PROBLEM SHAPE (don't use write-back for computed values).
