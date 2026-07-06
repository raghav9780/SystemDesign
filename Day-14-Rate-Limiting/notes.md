# Day 14 — Rate Limiting
**Phase 1: Foundations** | Score: 8/10 | Hands-on topic (algorithms taught + implementation built)

> **One-sentence idea:** Rate limiting = **capping how many requests a client can make in a time window**, so
> one caller can't overwhelm the system, run up your bill, or starve everyone else. It's the **bouncer counting
> how fast people walk in** — not *who* they are (that's auth), just *how often* they knock.

## 1. Why rate limiting exists (name these — interviewers ask "why?")
1. **Protect resources from overload** — a spike or a buggy retry-loop client can exhaust CPU/DB connections and
   take the service down for *everyone*; the limiter sheds excess load before it hits the fragile parts.
2. **Prevent abuse** — brute-force logins, credential stuffing, scraping, spam; capping attempts/IP makes attacks
   impractical.
3. **Fairness / multi-tenancy** — on a shared API, one greedy tenant shouldn't eat all capacity.
4. **Cost control** — if each request costs money (LLM call, 3rd-party API, egress), a runaway client bankrupts you.
5. **Cascading-failure prevention** — rate limiting is a form of **load shedding**: better to cleanly reject 5%
   with `429` than let 100% degrade into timeouts (Day 21 Resilience).

## 2. Where the limiter lives
- **Client-side** — polite but untrustworthy (attacker removes it). Never the only defense.
- **API Gateway / reverse proxy** (most common) — the natural chokepoint every request already flows through
  (Day 12/13). Nginx, Kong, AWS API Gateway, Envoy.
- **In the service / middleware** — when the limit needs app logic the gateway can't see ("10 free reports/month").
- ⭐ **Counter state MUST live in a shared store (Redis), not one server's memory.** Behind an LB with 8 servers,
  a local counter gives each client **8× its limit** (one counter per server). *State lives in a SHARED store,
  never on the server* — the signature lesson, again.

## 3. What do you limit *by* (the key)?
- **API key / user ID** — best for authenticated APIs; limit follows the account.
- **IP address** — for anonymous traffic; caveat: NAT/carriers share one IP (throttles a whole office), and
  attackers rotate IPs.
- **Combination** — `(user_id, endpoint)` so `/search` and `/upload` have separate limits.
- Counter key looks like `ratelimit:{user_id}:{endpoint}`.

## 4. ⭐ The five algorithms

### 4.1 Fixed Window Counter
Divide time into fixed buckets (each calendar minute); count in the current bucket; reject over the limit; reset
at the next bucket. ✅ dead simple, 1 int/client. ❌ **boundary-burst problem** ⭐ — 100 reqs at `12:04:59` + 100
at `12:05:00` = **200 in one second**, double the rate, both windows individually legal.

### 4.2 Sliding Window Log
Store a **timestamp per request**; on each request drop timestamps older than `now-60s` and count the rest.
✅ **perfectly accurate**, no boundary burst. ❌ **memory-expensive** — one entry per request (10k req/min = 10k
entries/client). Prohibitive at scale.

### 4.3 Sliding Window Counter (practical favorite) ⭐
Approximate the log cheaply: keep current-window count + previous-window count, weight previous by its overlap
fraction. `rate = current + previous × (overlap fraction)`. E.g. 30s into a 60s window → `current + previous×0.5`.
✅ smooths the boundary problem with only **2 ints/client** — near-log accuracy at fixed-window cost (Cloudflare
uses this). ❌ slightly approximate (assumes even distribution), error tiny in practice.

### 4.4 Token Bucket ⭐⭐ (most important to implement)
Bucket holds up to **N tokens**; tokens **refill at a steady rate**; each request **takes one token**; empty →
reject. A quiet client **accumulates** tokens and can fire a **burst up to N**, then is throttled to the refill
rate. Two params: **capacity** (max burst) + **refill_rate** (sustained throughput). ✅ **allows controlled
bursts**, 2 numbers/client, smooth (AWS, Stripe). ❌ a bit more logic (lazy refill).

### 4.5 Leaky Bucket
Requests enter a **queue**; server processes at a **constant leak rate**; queue full → overflow/drop. Smooths
output to a constant rate (bursts are queued & released steadily). ✅ perfectly smooth outflow (protects a
burst-intolerant downstream). ❌ adds latency (requests wait); bursts delayed, not served immediately.

⭐ **Token vs Leaky bucket:** both cap sustained rate. **Token bucket ALLOWS bursts** ("save up and spend",
favors client). **Leaky bucket FORCES smooth constant output** ("steady drip", favors downstream).

| Algorithm | Memory/client | Bursts? | Accuracy | Verdict |
|---|---|---|---|---|
| Fixed window | 1 int | edge-burst bug | low | simplest, flawed |
| Sliding log | N timestamps | no | exact | accurate, costly |
| Sliding counter | 2 ints | no | ~exact | **best general default** |
| Token bucket | 2 numbers | **yes (controlled)** | exact | **best when bursts OK** |
| Leaky bucket | queue | no (smoothed) | exact | steady downstream |

## 5. ⭐ Implementation (Rule F)

### 5.1 Token bucket in-memory — **lazy refill** is the trick
Don't run a background timer; on each request compute how many tokens *should* have been added since last check.
```python
import time
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity          # max tokens = max burst
        self.refill_rate = refill_rate    # tokens/sec = sustained rate
        self.tokens = capacity            # start full
        self.last_refill = time.time()
    def allow(self):
        now = time.time()
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)  # lazy refill
        self.last_refill = now
        if self.tokens >= 1:
            self.tokens -= 1              # take a token → allow
            return True
        return False                     # empty → reject (429)
```
State = just `tokens` + `last_refill`. **Why lazy, not a timer:** computing tokens on-demand from elapsed time is
one line of math per request — no per-client background thread, so it scales to millions of keys.

### 5.2 ⭐ Distributed token bucket — Redis + Lua (production)
In-memory breaks behind an LB (each server = own bucket → N× limit). Fix: **store state in Redis** so all servers
share it. New danger: two servers read-modify-write the same key concurrently → **race condition** (both read
1 token, both allow, tokens go negative). Fix: **atomicity via a Lua script** — Redis runs a script to completion
without interleaving anything else.
```lua
-- KEYS[1]=bucket key; ARGV=capacity, refill_rate, now(sec), cost
local cap  = tonumber(ARGV[1]); local rate = tonumber(ARGV[2])
local now  = tonumber(ARGV[3]); local cost = tonumber(ARGV[4])
local d = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(d[1]); local ts = tonumber(d[2])
if tokens == nil then tokens = cap; ts = now end          -- first request: start full
local elapsed = math.max(0, now - ts)
tokens = math.min(cap, tokens + elapsed * rate)           -- lazy refill
local allowed = 0
if tokens >= cost then tokens = tokens - cost; allowed = 1 end
redis.call('HMSET', KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('EXPIRE', KEYS[1], math.ceil(cap / rate) + 1)  -- auto-clean idle keys
return allowed
```
⭐ Whole script runs atomically → **no two servers interleave → race killed.** `cost` param lets expensive
requests spend more tokens. `EXPIRE` (≈ full-refill time) evaporates idle keys → no memory leak; a returning user
re-inits full. *(Simpler `INCR key` + `EXPIRE key 60` exists for fixed-window, but INCR has an atomicity gap
between increment and expiry, and it's the flawed algorithm — Lua + token bucket is the robust answer.)*

### 5.3 Client-facing contract (a scored sub-part — don't skip)
Reject with **HTTP `429 Too Many Requests`** + headers so good clients back off correctly:
- **`Retry-After: 30`** — seconds to wait. **Why it helps:** tells the client exactly when to retry, so it stops
  hammering with retries that keep getting rejected.
- **`X-RateLimit-Limit` / `-Remaining` / `-Reset`** — the cap, calls left, window-reset time → lets a client
  self-throttle *before* hitting the wall.

## 6. Advanced probes
- **Tiered limits** — free 100/hr vs pro 10k/hr; limit looked up per API-key plan.
- **Allow-lists** — internal services/partners bypass limits.
- **Local + distributed hybrid** — small per-server in-memory limiter absorbs the common case; Redis for the
  shared global count → less Redis load.
- ⭐ **Fail-open vs fail-closed** — if Redis is down, **allow all** (fail-open, favors availability) or **reject
  all** (fail-closed, favors protection)? For a rate limiter, usually **fail-open** — a limiter outage shouldn't
  take down your whole API — but it's a conscious trade-off (see follow-up).

---

## 🎯 Tasks & my answers

**Task 1 — Design (public API: free 100/hr, paid 10k/hr, 8 servers behind an LB):**
1. **Token bucket** — handles bursts, costs only 2 numbers/client. ✅ (2/2)
2. Local per-server counter → each server counts separately, so a free user gets **100/hr × 8 servers = 800/hr
   (8× the limit)**. Fix: a **shared Redis counter** so all 8 servers share one count. *(I said "serving the same
   by different server / wrong data" — correct idea, but didn't quantify the 8× consequence.)* (1.5/2)
3. Two servers hit the shared counter the same millisecond = a **race condition**; make the check atomic with a
   **Lua script** (Redis runs it to completion, no interleaving). ✅ (named mechanism + problem)
4. **`429 Too Many Requests`** + **`Retry-After: 30`** — *why:* tells the client exactly when to retry so it
   stops spamming rejected retries. *(I wrote "after 30s or x-ratelimit-limit" — muddled two headers and skipped
   the why.)* (0.5/1)

**Task 2 — Hands-on (built the distributed Lua token bucket above, §5.2 — my own work, correct & complete):**
- capacity + refill_rate params, `tokens`+`ts` state ✅
- lazy refill with `max(0,…)` and `min(cap,…)` clamps ✅
- conditional decrement by `cost`, `HMSET` back ✅
- `EXPIRE math.ceil(cap/rate)+1` cleanup ✅ (correct; tighter than the ×2 I was shown, still safe)
- ⚠️ Wrote the code but skipped the prose the steps asked for: **why lazy** (no per-client timer → scales),
  **why atomic** (Redis serializes the script → two servers can't both allow), **why EXPIRE** (idle keys would
  leak memory). Code was right; the *spoken* answers were the missing sub-parts.

**Net:** knowledge ~9.5; score 8 purely from unspoken "name it / why" clauses. Recurring gap, not a knowledge gap.

---

## ⭐ Follow-up (RESOLVED)
**Q:** Redis (the shared limiter store) goes **down**. (1) What happens to the API if the limiter can't reach
Redis and it's unhandled? (2) Name **both** design choices for this failure, say which you'd pick for a *rate
limiter* and **why**. (3) What's the **term** for this design decision?

**(1)** Unhandled, every request that hits the dead Redis call fails/hangs → **Redis becomes a single point of
failure (SPOF) that takes the whole API down.** *(Precise framing: the limiter itself breaks, so requests never
get processed — not that the limiter "rejects" them.)*

**(2) fail-open vs fail-closed → I'd pick fail-open** for a rate limiter: a *limiter* outage must not cause an
*API* outage. You briefly lose protection (clients could exceed limits during the blip) — an acceptable trade vs.
full downtime. **Mitigation:** a small **local in-memory fallback limiter** per server so you're not totally
unprotected while Redis is out. (A gate where over-admitting is catastrophic — e.g. paid metering — might instead
choose **fail-closed**.) ✅

**(3)** Two terms apply: **fail-open vs fail-closed** (the specific name for "what to do when the dependency
dies") and, generally, **graceful degradation** — when a component fails, drop to a reduced-but-working mode
rather than fall over completely. Fail-open *is* graceful degradation applied to a limiter. (Returns Day 21,
Resilience.) *(I named fail-open + the trade-off but skipped naming the term in part 3 — the recurring
answer-every-sub-part gap, third day running.)*

---

## Personal Notes (recurring gaps)
- [ ] ⭐ **#1 GAP — answer every "name it / why" clause.** ALL 4 lost points today were unspoken sub-parts:
  the 8× consequence (1.2), the header's purpose (1.4), why-lazy / why-atomic / why-expire (Task 2). The code
  proved I know it; the score fell because I didn't *say* it. A "why" or "name the specific X" is a scored box.
- [x] **Naming precision** ✅ — "token bucket", "Lua script for atomicity", "race condition" all landed cleanly.
- [x] **Implementation** ✅✅ — wrote a correct distributed atomic token bucket in Lua unprompted (senior-level),
  incl. a `cost` param and TTL cleanup.
