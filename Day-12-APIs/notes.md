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
- **Safe** = read-only, no server change: `GET` (also `HEAD` — GET without the body — and `OPTIONS`).
- **Idempotent** = N calls == 1 call in effect: `GET`, `HEAD`, `OPTIONS`, `PUT`, `DELETE` (delete twice → still gone;
  put sets a fixed state). Every safe method is automatically idempotent.
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

## ⭐ Interview-completeness additions (audit pass)

### A. (CRITICAL) Client-push transports: short polling vs long polling vs SSE vs WebSockets
Plain request/response has one hard limit: **the client always initiates.** The SERVER can't push. §9's webhook solves server→*server*; this solves server→*browser/app*. Four options, cheapest → richest:
- **Short polling** — client asks every N seconds ("new data?"). Dead simple, works everywhere, but **wasteful** (mostly empty responses) and **laggy** (latency ≈ the poll interval).
- **Long polling** — client sends a request; the server **HOLDS it open** until data arrives (or a timeout), then responds; client **immediately re-requests.** Near-real-time over plain HTTP, no special protocol. Cost: **ties up a connection per client**, and reconnect storms are bursty.
- **SSE (Server-Sent Events)** — ONE long-lived HTTP connection; server **streams events** (`Content-Type: text/event-stream`); **auto-reconnect is built in.** But **one-directional (server→client only)** and **text only.** Great for feeds, notifications, live scores, progress bars.
- **WebSockets** — a single TCP connection **upgraded** (`Upgrade: websocket`) to **FULL-DUPLEX**; both sides push anytime. Best for **chat, multiplayer games, collaborative editing, trading**. Cost: **stateful, long-lived connections** → harder to load-balance (needs **sticky routing** or a **shared pub/sub backplane** like Redis so any server can reach any client).
- ⭐ **Decision:** server→client only + HTTP-friendly → **SSE**; true two-way low-latency → **WebSockets**; can't change infra / firewall-hostile environment → **long polling**.

### B. Webhook reliability engineering (§9 only *defines* push-vs-pull — this is the hard part)
- **Delivery is at-least-once** → the receiver **MUST be idempotent** (dedupe on the event ID — same idea as §3, applied to inbound events).
- **Retries:** exponential backoff **+ jitter**, a **capped** number of attempts, then drop to a **DLQ / failed-deliveries dashboard** for manual replay.
- **Security — sign the payload:** server computes an **HMAC** of the body with a **shared secret** and puts it in a header (e.g. `Stripe-Signature`); the receiver **recomputes and compares.** Include a **timestamp** in the signed data and **reject old signatures** → stops replay attacks.
- **Ordering is NOT guaranteed** → tolerate out-of-order delivery using **event timestamps / version numbers**, not arrival order.

### C. HATEOAS + the Richardson Maturity Model (name it, even if rarely built)
- **HATEOAS** = *Hypermedia As The Engine Of Application State*: responses include **links to the next possible actions**, so the client **discovers transitions** instead of hardcoding URLs. E.g. an order returns `"_links": { "pay": "/orders/42/pay", "cancel": "/orders/42/cancel" }`.
- This is **Richardson Level 3** — the maturity ladder: **L0** single endpoint/RPC-over-HTTP · **L1** resources · **L2** verbs + resources + status codes · **L3** hypermedia (HATEOAS).
- ⭐ **Reality check worth saying out loud:** almost nobody implements L3 fully — **most "REST" APIs are really Level 2** (proper nouns + verbs + status codes). Naming the model signals you know where the line actually is.

### D. Input validation & contract hygiene (validate at the edge)
- **Validate at the edge** (gateway/handler) before any business logic: **schema-validate the body** (OpenAPI / JSON Schema for REST, protobuf for gRPC), **reject unknown or oversized payloads**, and return **`400`** with a **structured error listing the failed fields** (not a bare string).
- **Naming/shape best practices:** **plural nouns** (`/users` not `/user`), **consistent casing**, **filter/sort via query params** (`?status=active&sort=-created`), **always paginate collections**, and on create return the **created resource + a `Location` header** with `201`.
- **Never break a published contract** — **version instead** (§8). A silent breaking change is the cardinal API sin.

### E. Small fixes to earlier sections
- **`204 No Content`** — add to §2's status-code list: the canonical response for a successful **`DELETE`** (and empty-body **`PUT`**) where there's nothing to return.
- **`PATCH` is NOT inherently idempotent** — correction to §2's safe/idempotent list: unlike `PUT`/`DELETE`/`GET`, whether `PATCH` is idempotent **depends on the patch** (e.g. "set field=x" is idempotent; "increment by 1" is not).
- **Gateways can do PROTOCOL TRANSLATION** — add to §7: expose **public REST** at the door while calling **internal gRPC** services behind it (also fronts the gRPC-Web problem from §5).
- **GraphQL authz is FIELD-LEVEL** — add to §4's 👎: because one query can touch many fields/resolvers, you must authorize **per field**, which is harder than REST's simple **per-route** checks.

---

## ⭐ Completeness additions (pass 2)

### F. HTTP status codes — the taxonomy, explained (not just listed)
**The idea:** the first digit tells the client **whose problem it is and what to do next** — that's why the right code is
part of the contract. A client's retry logic, a load balancer's health checks, and your dashboards all key off it.
- **2xx — it worked.**
  - `200 OK` — success with a body. `201 Created` — a new resource exists; return it **plus a `Location` header** pointing
    to it. `202 Accepted` — "got it, I'll do it **asynchronously**" (e.g. a video upload queued for processing — Day 11);
    return a status URL to poll. `204 No Content` — success, nothing to return (typical `DELETE`).
- **3xx — look elsewhere.** `301`/`308` moved permanently, `302`/`307` temporarily, `304 Not Modified` — "your cached copy
  (matched by `ETag`) is still fresh, I'm not resending it" (Day 3/4 caching).
- **4xx — the CLIENT must change something; retrying the same request won't help** (except `429`).
  - `400 Bad Request` — malformed: invalid JSON, wrong types, missing required field.
  - ⭐ **`401 Unauthorized` vs `403 Forbidden`** (the classic trap — the names are misleading):
    - **401 = "I don't know who you are."** No token, expired token, bad signature → **authentication** failed. Fix: log in /
      refresh the token. (The server should send a `WWW-Authenticate` header saying how to authenticate.)
    - **403 = "I know exactly who you are, and you're not allowed."** Valid token, but a regular user calling an admin
      endpoint → **authorization** failed. Logging in again won't help.
    - Memory hook: 401 → *authentication* (who), 403 → *authorization* (what you may do). Some APIs return `404` instead of
      `403` to avoid revealing that a private resource even exists.
  - `404 Not Found` — no such resource. `405 Method Not Allowed` — e.g. `DELETE` on a read-only collection.
  - ⭐ **`409 Conflict`** — the request is valid but **clashes with the current state** of the resource: the seat is already
    booked, a username is taken, an optimistic-lock version mismatch, or (Task 2 above) the same idempotency key is still
    in progress. Client must re-read state and decide.
  - **`412 Precondition Failed`** — the client sent `If-Match: <etag>` ("only update if nobody changed it since I read it")
    and somebody did. HTTP's built-in optimistic concurrency (Day 5).
  - **`422 Unprocessable Content`** — the JSON is well-formed, but the *meaning* is invalid: `"quantity": -3`, end date before
    start date. Many APIs use `400` for both — pick one convention and keep it consistent; what matters is the structured
    error body (G).
  - ⭐ **`429 Too Many Requests`** — rate limited (Day 14). Include **`Retry-After`** (seconds, or a date) so well-behaved
    clients know exactly when to come back. The one 4xx that *is* meant to be retried — later.
- **5xx — the SERVER (or something behind it) failed; the client may retry, with backoff, if the operation is idempotent.**
  - `500 Internal Server Error` — an unhandled bug/exception.
  - ⭐ **`502 Bad Gateway`** — a gateway/proxy (Day 13) reached the upstream service but got an **invalid** response (it
    crashed mid-response, closed the connection, sent garbage).
  - ⭐ **`503 Service Unavailable`** — the service is **temporarily** unable to handle requests: overloaded, shedding
    load, in maintenance, no healthy instances. Often with `Retry-After`.
  - ⭐ **`504 Gateway Timeout`** — a gateway/proxy **waited too long** for the upstream and gave up. Key subtlety: the
    upstream may **still finish** the work → a blind retry of a non-idempotent `POST` can double-execute it → this is
    exactly why idempotency keys (§3) exist.
  - Mnemonic for 502/503/504: **502 = got a bad answer, 503 = nobody can take your call right now, 504 = no answer in time.**
- **Retry rule of thumb:** don't retry 4xx (fix the request) except `429` (wait `Retry-After`); retry `502/503/504` (and
  network errors) with exponential backoff + jitter (Day 11 §13), only for idempotent operations or with an idempotency key.

### G. A structured error body — what to actually return
Never `200 OK` with `{"error": …}` inside, and never a bare string. A standard shape exists: **RFC 9457 "Problem Details for
HTTP APIs"** (it replaced RFC 7807), served as `Content-Type: application/problem+json`:
```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Request validation failed",
  "status": 422,
  "detail": "2 fields are invalid.",
  "instance": "/orders/req-7f3a91",
  "errors": [
    { "field": "quantity",  "message": "must be at least 1" },
    { "field": "ship_date", "message": "must be in the future" }
  ],
  "request_id": "req-7f3a91"
}
```
- `type` — a stable URI identifying the **kind** of error (clients branch on this, not on the human text).
- `title` — short human summary of that kind; `status` — repeats the HTTP code; `detail` — specifics for *this*
  occurrence; `instance` — which occurrence (useful in logs).
- Extra members are allowed by the RFC: `errors` (per-field list for forms) and a `request_id`/trace ID so support
  can find the exact request in logs (Day 22 observability).
- Rules: **machine-readable code + human message**, consistent across every endpoint, and **no stack traces or SQL**
  in production responses (leaks internals — Day 23).

### H. API Gateway vs BFF — explicitly different jobs
- **API Gateway = ONE front door for ALL clients.** It handles **cross-cutting, client-agnostic** concerns: TLS termination,
  authentication, rate limiting, routing `/payments/*` to the payment service, logging. It should stay **thin** — no
  business logic, no per-screen data shaping.
- **BFF (Backend-for-Frontend) = one small backend PER client type** (web BFF, iOS BFF, Android BFF), usually **owned by that
  frontend's team**. It does **client-specific** work: calls 4 services, merges the results, trims fields the mobile
  screen doesn't need, formats for that UI. Why separate per client: the web dashboard wants rich data in one call; the
  mobile app on a slow network wants a tiny payload — one shared API ends up serving neither well, and every frontend
  team blocks on one backend team.
- **How they combine:** `client → API Gateway (auth, rate-limit, TLS) → the right BFF → microservices`. Gateway = generic
  plumbing for everyone; BFF = tailoring for one.
- **BFF vs GraphQL:** both solve "different clients want different shapes". GraphQL lets the **client** declare the shape
  against one schema; a BFF puts that shaping in **server code** per client. Teams often choose one or the other (or a
  GraphQL server *as* the BFF).
- And the cousins (Day 13): **load balancer** = spreads traffic across identical instances; **reverse proxy** = sits in front
  of servers (TLS, caching, compression, routing); **API gateway** = an API-aware reverse proxy (auth, keys, quotas,
  versioning, transformations). Same position in the diagram, increasing application awareness.

### I. 🛠️ Webhook signature verification — the code (Stripe's scheme)
§B said "verify the HMAC". Here is exactly how, following Stripe's documented scheme. The `Stripe-Signature` header looks
like `t=1492774577,v1=5257a869…` — a timestamp plus one or more signatures. Stripe computes **HMAC-SHA256** with your
endpoint secret over the string **`"{timestamp}.{raw_body}"`**.
```python
import hmac, hashlib, time

def verify_webhook(raw_body: bytes, sig_header: str, secret: str, tolerance_sec: int = 300) -> bool:
    pairs = [item.split("=", 1) for item in sig_header.split(",")]
    timestamp = next(v for k, v in pairs if k == "t")
    signatures = [v for k, v in pairs if k == "v1"]          # ignore other schemes (prevents downgrade)

    signed_payload = timestamp.encode() + b"." + raw_body      # "t.body" — exactly what the sender signed
    expected = hmac.new(secret.encode(), signed_payload, hashlib.sha256).hexdigest()

    if abs(time.time() - int(timestamp)) > tolerance_sec:      # too old → possible replay attack
        return False
    return any(hmac.compare_digest(expected, s) for s in signatures)   # constant-time comparison
```
Line by line, and **why**:
- **Parse the header** into the timestamp `t` and the `v1` signatures (there can be several while a secret is being rotated).
- **Use the RAW body bytes.** If your framework parses and re-serializes the JSON (reordering keys, changing whitespace),
  the bytes differ and the HMAC won't match. Verify first, parse after.
- **Sign `timestamp + "." + body`.** Because the timestamp is *inside* the signed data, an attacker can't change it without
  breaking the signature.
- **Reject old timestamps** (Stripe's libraries default to a **5-minute** tolerance) → a captured, valid request can't be
  replayed tomorrow. Keep server clocks in sync (NTP).
- **`hmac.compare_digest`** compares in **constant time**. A normal `==` stops at the first differing character, and an
  attacker measuring response times could guess the signature byte by byte (a *timing attack*).
- After verifying: **dedupe on the event ID** (§B — delivery is at-least-once), **return `2xx` fast**, and do the real work
  asynchronously via a queue (Day 11) so a slow handler doesn't cause timeouts → retries → more duplicates.
- On the **sending** side (if you build webhooks): sign the same way, retry failed deliveries with exponential backoff over
  hours/days (Stripe retries for up to 3 days in live mode), then DLQ/dashboard for manual replay.

### J. Resource modeling — the rules, with examples
1. **Nouns, not verbs.** `POST /orders`, not `POST /createOrder`. The HTTP method is the verb.
2. **Plural collections, IDs for items:** `/orders` (the collection) → `/orders/42` (one item).
3. **Nest only for true ownership, and only one level deep:** `/users/42/orders` ("orders belonging to user 42") is fine;
   `/users/42/orders/7/items/3/reviews` is not — once an item has its own ID, address it directly: `/reviews/99`.
4. **Path params identify; query params filter.** Path = *which* resource (required): `/orders/42`. Query = *how* you want the
   collection (optional): `/orders?status=shipped&sort=-created_at&limit=20`.
5. **Actions that aren't plain CRUD:** model them as a **state change** or a **sub-resource**:
   - `PATCH /orders/42 {"status": "cancelled"}`, or
   - `POST /orders/42/cancellation` (creates a "cancellation" — handy when cancelling has its own data: reason, refund).
   - Pragmatic fallback some big APIs use: `POST /orders/42/cancel` — acceptable if consistent, but know it's a verb.
6. **Model what the client needs, not your DB tables.** A "booking" resource may span 3 internal tables; clients shouldn't care.

### K. GraphQL's N+1 problem and DataLoader — worked example
- **The setup:** query `{ posts(limit: 10) { title author { name } } }`. The `posts` resolver runs **1** query for 10 posts.
  Then the `author` resolver runs **once per post** → **10 more** queries (`SELECT * FROM users WHERE id = ?` × 10).
  That's **1 + N** queries — and with nested lists (posts → comments → authors) it multiplies.
- **Why GraphQL invites it:** each field's resolver is written independently ("given a post, fetch its author"), so the
  framework naturally calls it once per item.
- **DataLoader fix (batching + caching per request):**
  1. Each `author` resolver calls `userLoader.load(post.author_id)` instead of querying — it gets back a promise.
  2. DataLoader **waits until the current tick of resolvers has run**, collecting all requested IDs: `[7, 3, 7, 12, …]`.
  3. It **deduplicates** them and issues **one** batched query: `SELECT * FROM users WHERE id IN (3, 7, 12, …)`.
  4. It hands each resolver its matching user. Result: **2 queries total** instead of 11.
  - Its cache lives for **one request** only (so it never serves stale data across users).
- Related GraphQL protections: **query depth / complexity limits** (a malicious deeply-nested query can be very expensive),
  and **persisted queries** (clients send a hash of a pre-registered query, often via `GET` — which also brings back HTTP caching).

### L. Versioning — what counts as "breaking", and how to retire versions
- **Non-breaking (safe without a new version):** adding a new endpoint, adding an **optional** request field, adding a
  new **response** field (clients must ignore unknown fields — say so in your docs).
- **Breaking (needs a new version):** removing/renaming a field, changing a type (`"price": "9.99"` → `999`), making an
  optional field required, changing error codes or the meaning of a value.
- **Strategies:** URL path `/v2/orders` (most visible, easiest to route at the gateway — the common default); header
  (`Accept: application/vnd.example.v2+json`) — cleaner URLs, harder to test in a browser; query param `?version=2`.
  Stripe's well-known variant: **date-based versions** pinned per account and overridable per request with a
  `Stripe-Version` header — they keep old behaviour running for old integrations.
- **Retiring a version:** announce, add `Deprecation`/`Sunset` response headers with the shutdown date, monitor who still
  calls v1 (gateway metrics), contact them, then turn it off.

### M. Pagination — the "why" behind offset's slowness, and real cursors
- **Why offset is slow on deep pages:** `LIMIT 20 OFFSET 1000000` makes the DB **find and throw away 1,000,000 rows** before
  returning 20. Cost grows with the page number. Keyset/cursor (`WHERE id < :last_id ORDER BY id DESC LIMIT 20`) jumps
  straight to the spot via the index — page 50,000 costs the same as page 1.
- **Real cursors need a unique tie-breaker:** sorting by `created_at` alone breaks when two rows share a timestamp (one may be
  skipped or repeated). Sort by `(created_at, id)` and filter `WHERE (created_at, id) < (:last_created_at, :last_id)` —
  backed by an index on `(created_at, id)` (Day 10 composite index).
- **Make the cursor opaque:** return `"next_cursor": "eyJjIjoiMjAyNi0xMC0yN1QxMDowMDowMFoiLCJpIjo5ODd9"` (base64 of the last
  `(created_at, id)`) rather than raw values, so clients can't build or depend on it and you can change its contents later.
- Trade-off to state: cursor pagination **can't jump to "page 37"** — fine for feeds/infinite scroll, not for numbered admin tables.

### N. Authentication vs authorization at the API (preview — full treatment Day 23)
- **Authentication (authn) = who are you?** **Authorization (authz) = what may you do?** (→ 401 vs 403, section F.)
- **API keys** — a long random secret identifying a calling *application*; simple, typical for server-to-server. Send in a
  header, never in the URL (URLs end up in logs).
- **JWT (JSON Web Token)** — a signed token carrying claims (`user_id`, roles, expiry). Any server can **verify the signature
  without a DB lookup** → stateless (fits §2's statelessness). Downside: hard to revoke before expiry → keep them short-lived
  and use refresh tokens.
- **OAuth 2.0** — **delegated** access: "let this app read my Google Calendar" without giving it your password; the app
  receives a scoped **access token**.
- **RBAC (Role-Based Access Control)** — permissions attached to roles (`admin`, `editor`, `viewer`); the authz check is
  "does this user's role allow this action on this resource?"
- Where: the **gateway** usually does authentication once (verify the token); **services** still do fine-grained
  authorization ("is this *your* order?") because only they know resource ownership.

### O. Worked example — a Ticketmaster-style booking API (high contention)
The classic interview prompt that combines everything above. Thousands of users want the same few seats at once.
```
GET  /events/{event_id}/seats?section=A          → 200, seat map with status (available / held / sold)
POST /events/{event_id}/holds                    → reserve seats for a few minutes
     Idempotency-Key: 7c1e…                        body: {"seat_ids": ["A-12", "A-13"]}
     → 201 Created  {"hold_id": "h_981", "expires_at": "…+10min"}
     → 409 Conflict {"type": ".../seat-unavailable", "unavailable": ["A-13"]}
POST /holds/{hold_id}/purchase                   → pay and convert the hold into a booking
     Idempotency-Key: 91ab…                        body: {"payment_method": "pm_…"}
     → 201 Created  {"booking_id": "b_5521"}
     → 410 Gone (hold expired) / 409 (already purchased) / 402 (payment failed)
DELETE /holds/{hold_id}                          → 204, release seats early
```
- **Why a two-step hold → purchase:** paying takes time (card forms, 3-D Secure). A **temporary hold with an expiry** stops two
  people paying for the same seat; if the buyer walks away, the hold expires and the seats return automatically.
- **Contention handled in the data layer, reported via the API:** the hold succeeds only if an atomic conditional update
  succeeds (`UPDATE seats SET status='held', hold_id=… WHERE seat_id IN (…) AND status='available'` and check the row count,
  or a Redis `SET seat:A-12 h_981 NX EX 600` per seat) → anyone who loses the race gets **`409 Conflict`**.
- **Idempotency keys** on both POSTs: a retried "purchase" after a timeout (a `504`!) must never charge twice.
- **`429` + a virtual waiting room** at the gateway for on-sale spikes (Day 14 rate limiting).

## Personal Notes (recurring gaps)
- [ ] ⭐ **#1 GAP, 4 days running — NAME the concept, don't describe/contradict it.** My *implementation* of
  idempotency was 4/4 (near-perfect), yet I lost design points by not saying "idempotency key" and by saying "or
  the server itself" (contradicting the shared-store rule I know). Same with pagination: I built cursor pagination
  but called it "offset." **The knowledge is 10/10; the labeling costs the points.** Drill: narrate the nouns of
  your own design out loud.
- [ ] **Shared store, never the server** — said "stateless cache" (right) then "or the server itself" (wrong). Don't undercut it.
- [x] **Idempotency implementation = senior-level** ✅ (PROCESSING sentinel, atomic SET NX, replay, cleanup, TTL reasoning).
- [x] **REST/gRPC style choice + gateway jobs** ✅.
