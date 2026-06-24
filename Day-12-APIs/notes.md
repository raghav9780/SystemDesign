# Day 12 — APIs (REST, GraphQL, gRPC, Gateways, Idempotency)
**Phase 1: Foundations** | Score: 9/10

> **One-sentence idea:** An API is a **contract** — how one piece of software asks another for something
> (what you can request, how, and what comes back) without either knowing the other's internals. Like a
> **restaurant menu**: you order from a fixed menu (the contract), the kitchen (server) decides how to cook;
> as long as the menu is stable, the kitchen can be rebuilt entirely and you never notice. Today = the 3
> menu styles (REST, GraphQL, gRPC), the rules that make an API safe to retry (idempotency), and the front
> door in front of them all (API gateway).

## 1. What an API is — the contract
- Two services talk via an agreed interface: "send a request shaped like THIS, get a response shaped like THAT."
- Point = **decoupling**: the client depends on the *contract*, not the implementation. Server can change
  language/DB/host freely; clients keep working while the contract holds.
- Dominant transport = **HTTP**, so most of this is "use HTTP well."

## 2. REST — the dominant style
- **Resources are nouns, addressed by URL:** `/users`, `/users/42`, `/users/42/orders`. Name *things*, not
  actions (`/getUser` is an anti-pattern — the verb goes in the HTTP method).
- **HTTP verbs = actions:** `GET` read · `POST` create · `PUT` replace whole · `PATCH` partial update · `DELETE` remove.
- **Status codes = outcome:** `2xx` ok (200, 201 Created) · `4xx` client's fault (400, 401, 403, 404, 409, 429)
  · `5xx` server's fault (500, 503). Using the right code IS part of the contract.

### ⭐ Statelessness (REST principle + the signature lesson)
- Each request carries **everything** the server needs (auth token, params); the server keeps **no per-client
  memory between requests**. Why: if the session lived in one server's memory, your next request would have to
  hit *that exact server* → breaks load balancing + horizontal scaling. Instead **state lives in a shared store**
  (DB/Redis) → any server handles any request. This is *why* REST is stateless.

### Safe vs idempotent methods (interview favorite)
- **Safe** = read-only, no server change: `GET`.
- **Idempotent** = N calls == 1 call in effect: `GET`, `PUT`, `DELETE` (delete twice → still gone; put sets a
  fixed state).
- **NOT idempotent: `POST`** — each call creates a new thing → "create order" twice = two orders. §3 fixes this.

## 3. ⭐ Idempotency & idempotency keys (must-know practical)
- **Problem:** client `POST /payments` $100, network drops the *response* (not the request) → client can't tell
  "request lost" from "response lost" → retries → **double charge.** Unavoidable in distributed systems.
- **Fix — idempotency key:** client generates a unique UUID for the *logical operation*, sends header
  `Idempotency-Key: abc-123`. Server: (1) first time → process, **store key + result**, return; (2) retry (same
  key) → **return the stored result**, don't reprocess.
- ⭐ Client OWNS the key (so a retry reuses it); server OWNS the dedup (stores seen keys in **Redis/DB**, with a TTL).
- Same idea as Day 11's consumer-side idempotency, applied to HTTP. How Stripe/PayPal work.

## 4. GraphQL — client decides the data shape
- REST weakness: **over-fetching** (endpoint returns 30 fields, screen needs 2) + **under-fetching / N+1**
  (many round-trips to assemble nested data).
- **GraphQL (Meta):** ONE endpoint (`POST /graphql`); client sends a **query describing the exact shape** wanted,
  gets exactly that. Server defines a typed **schema**; each field has a **resolver** that fetches it.
- 👍 No over/under-fetch, one round-trip, strongly typed, great for varied clients (web vs mobile want different fields).
- 👎 (name these) **HTTP caching is hard** (all POST to one URL — can't cache by URL like REST GETs); expensive
  queries need depth limits; the **N+1 moves server-side** (naive resolver = 1 DB query/item → fix with **DataLoader**
  batching); more upfront complexity.

## 5. gRPC — fast, typed, service-to-service
- **Remote Procedure Call** style: call a remote function like a local one — `chargePayment(req)`.
- **Protocol Buffers (.proto):** define service + messages → generates client/server code in many languages;
  serialized as **compact binary** (not JSON) → smaller + faster to parse.
- **Runs on HTTP/2:** multiplexing (many calls/connection) + **streaming** (client/server/bidirectional).
- 👍 very high performance + low latency, strong typing, ideal for **internal microservice-to-microservice**.
- 👎 not human-readable (binary), **limited browser support** (needs gRPC-Web proxy), steeper curve. Rarely
  exposed directly to public browsers.

## 6. ⭐ REST vs GraphQL vs gRPC — when to use
| | **REST** | **GraphQL** | **gRPC** |
|---|---|---|---|
| Style | Resources + verbs | One endpoint, client query | Remote procedure calls |
| Format | JSON (text) | JSON | Protobuf (binary) |
| Best for | Public APIs, CRUD, simplicity | Varied clients, nested data, mobile | Internal microservices, low latency, streaming |
| Caching | **Easy** (HTTP GET) | Hard | Hard |
| Fetching | Over/under-fetch risk | Exactly what's asked | Exactly what's defined |
| Readable | Yes | Yes | No (binary) |
- ⭐ Rule of thumb: **REST** public/simple, **GraphQL** varied clients needing different slices of nested data,
  **gRPC** fast internal service-to-service. (NB: REST's win is **simplicity + universal support + caching**,
  NOT speed — gRPC is the fast one. Q1 fix.)

## 7. API Gateway — the single front door
- Single entry point in front of many services, handling **cross-cutting concerns** so each service doesn't:
  **routing** (`/payments/*`→payment svc) · **auth** (verify token once at the door, Day 23) · **rate limiting**
  (Day 14) · **SSL/TLS termination** · **request aggregation** (combine backend calls) · **caching/logging/metrics**.
- Keeps each microservice on business logic; gives clients one stable address.
- **BFF (Backend-for-Frontend):** a dedicated gateway per client type (mobile vs web) → tailored API each.

## 8. Cross-cutting design you must know
- **Versioning** — don't break old clients: **URI** (`/v1/users`, most common) · header · query param. Never
  silently break a contract.
- **Pagination** — never return millions of rows:
  - **Offset/limit** (`?page=3&limit=20`) — simple, but slow on deep pages and **shifts** if rows are inserted
    while paging → duplicates/skips.
  - ⭐ **Cursor/keyset** (`WHERE id < :last_id ORDER BY id DESC LIMIT 20`) — "give me N after this point." Stable
    under inserts, fast at scale → what large feeds use. **This is the right pick when rows are added constantly.**
- **Error handling** — right status code **and** a structured body (`{error, code}`), never 200-with-error-inside.
- **Auth (preview, Day 23)** — API keys (simple) or **bearer tokens/JWT** (users) in the `Authorization` header;
  keeps the API stateless.
- **Rate limiting (preview, Day 14)** — `429 Too Many Requests`.

## 9. Webhooks vs polling (push vs pull for APIs)
- **Polling** — client repeatedly asks "done yet?" → simple but wasteful.
- **Webhook** — client registers a URL; server **calls the client** when the event fires → efficient/real-time
  (API equivalent of Day 11's push).

## Real-system map
- Stripe/GitHub/Twitter public APIs → REST + idempotency keys + cursor pagination + webhooks.
- GitHub also offers GraphQL. Google/Netflix internal → gRPC. Kong / AWS API Gateway / Nginx → gateway layer.

---

## 🎯 Task 1 — Design (my answers)
1. **Style:** REST for the public app (simple, universal support, cacheable — NOT "fast"); **gRPC** internal
   (high perf, low latency). ✅
2. **Idempotency:** retry after signal loss may reprocess → use an **idempotency key** (client UUID in header);
   server dedups in a **shared store (Redis)** and **returns the stored response** on retry. ⚠️ I said "or the
   server itself" — WRONG, state must be in a shared store, never one server's memory (signature lesson).
3. **Gateway jobs:** SSL termination, auth, caching, rate limiting (also routing/aggregation). ✅
4. **Pagination:** I described `ORDER BY id DESC, use last id, LIMIT` — that IS **cursor/keyset** pagination (the
   correct pick, because trips are inserted constantly and offset would shift rows) — but I mislabeled it "offset."
   ⚠️ Name it **cursor pagination**; offset = page numbers and is the wrong fit here.

## 🎯 Task 2 — Hands-on: safe-to-retry `POST /rides` (idempotency key) — 4/4, senior-level ✅
**Request:**
```
POST /rides HTTP/1.1
Host: api.yourapp.com
Content-Type: application/json
Authorization: Bearer <token>
Idempotency-Key: a3f8c2d1-7b44-4e92-b6f0-1234abcd5678

{ "pickup": "12.9716,77.5946", "dropoff": "12.9352,77.6245", "rider_id": "usr_9988" }
```
**Handler logic (the key moves):**
```
key = headers["Idempotency-Key"]            // 400 if missing
stored = redis.get("idem:"+key)
if stored == "PROCESSING": return 409        // a retry while the original is mid-flight
if stored != null:        return deserialize(stored)   // replay the SAME response verbatim

// first time: claim the key ATOMICALLY before any work
claimed = redis.SET("idem:"+key, "PROCESSING", EX=30, NX=true)   // NX = only if absent
if not claimed: return 409                   // lost the check-then-act race → caller retries

try:
    ride = db.insertRide(...status="REQUESTED")
    response = {201, {ride_id, status, eta_mins}}
    redis.SET("idem:"+key, serialize(response), EX=86400)   // overwrite with final result, 24h TTL
    return response
catch e:
    redis.DELETE("idem:"+key)                // don't wedge the key on failure
    return 500
```
- ⭐ Why this is strong: the **`PROCESSING` sentinel + `SET NX EX`** is an *atomic claim* that closes the
  check-then-act race (two near-simultaneous retries → only one wins the `NX`, the other gets 409). I both NAMED
  the race and defended it in code.
- **TTL reasoning:** TTL = how long dedup protection lasts. Too short (30s) → a legit retry after expiry would
  reprocess (double-create); too long → storage grows. 24h is the industry norm.

---

## ⭐ Follow-up (RESOLVED) ✅
**Q:** Server hard-crashes (power loss, so `catch` never runs) in the window AFTER `db.insertRide` succeeds but
BEFORE `redis.SET(final response)`. Client retries at 5s and at 60s — duplicate-ride risk? Where must the REAL
idempotency guarantee live to be bulletproof?

**A:** Redis-only dedup is **not crash-safe** — the timeline shows the gap:
- **At 5s:** the `PROCESSING` key is still alive (set with `EX=30`) → retry gets **409 in-progress.** Safe.
- **At 60s:** the 30s `PROCESSING` TTL has **expired** and the crash meant the final response was never written →
  Redis is now **empty** for the key → the retry re-claims and calls `db.insertRide` **AGAIN**. This is the danger
  window Redis alone cannot cover.
- **The bulletproof fix lives in the DATABASE:** store the idempotency key as a column in `rides` with a **UNIQUE
  constraint** (or insert ride + key in one transaction). The duplicate insert at 60s violates the constraint →
  handler catches it, `SELECT`s the ride the first (pre-crash) request already committed, and returns THAT → no
  duplicate. **Redis = fast-path dedup for the common case; the DB unique constraint = the durable, crash-proof
  backstop / source of truth.** ✅ (I correctly named the DB idempotency key as the guarantee; refined the *why* —
  the 30s PROCESSING window expires and leaves a gap only the DB closes.)

---

## Personal Notes (recurring gaps)
- [ ] ⭐ **#1 GAP, 4 days running — NAME the concept, don't describe/contradict it.** My *implementation* of
  idempotency was 4/4 (near-perfect), yet I lost design points by not saying "idempotency key" and by saying "or
  the server itself" (contradicting the shared-store rule I know). Same with pagination: I built cursor pagination
  but called it "offset." **The knowledge is 10/10; the labeling costs the points.** Drill: narrate the nouns of
  your own design out loud.
- [ ] **Shared store, never the server** — said "stateless cache" (right) then "or the server itself" (wrong). Don't undercut it.
- [x] **Idempotency implementation = senior-level** ✅ (PROCESSING sentinel, atomic SET NX, replay, cleanup, TTL reasoning).
- [x] **REST/gRPC style choice + gateway jobs** ✅.
