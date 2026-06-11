# Day 1 — Scalability: Vertical vs Horizontal Scaling
**Phase 1: Foundations** | Score: 7/10

---

## Core Concept
- **Scalability** = system's ability to handle growth without falling over.

## Two Ways to Scale

### 1. Vertical Scaling (Scale UP ⬆️) — bigger machine
- Add more CPU / RAM / faster disk to ONE server.
- ✅ Simple — no code changes, no distributed complexity.
- ❌ Hard ceiling — a machine can only get so big.
- ❌ Single Point of Failure (SPOF) — that box dies, everything dies.
- ❌ Exponentially expensive at the high end.

### 2. Horizontal Scaling (Scale OUT ➡️) — more machines
- Add MORE servers, split work across them.
- ✅ Near-infinite scaling — keep adding nodes.
- ✅ Fault tolerant — one node dies, others keep serving.
- ❌ Complex — needs load balancer, data consistency, state handling.

## Key Mental Model
- Vertical = **easy but limited**. Horizontal = **hard but unlimited**.
- Real systems: start vertical (cheap/simple), go horizontal when they outgrow one machine.
- **Scale vertically FIRST** (instant band-aid, buys time), plan horizontal in parallel.
- The hard part of horizontal scaling = **STATE** (where does user data/session live?).

---

## Traps & Gotchas
- **DB trap:** Adding web servers but pointing them all at ONE database just MOVES the
  single point of failure to the DB. Web tier scaled, data tier still a bottleneck.
- **State trap:** User logs in on Server A → next request hits Server B → logged out.
  Fix: make web tier **stateless**, push sessions/data to a shared store.

---

## Solving the DB Bottleneck (order of effort)
1. **Cache** (Redis/Memcached) in front of DB — most traffic is reads. → Day 3
2. **Read replicas** — reads go to replicas, writes to primary. → Day 8
3. **Shard** — split data across multiple DBs (e.g. users A–M / N–Z). → Day 9
- Memorize: **Cache → Replicate → Shard**

## Storing Files (photos/videos)
- ❌ DON'T store large blobs in the database.
- ✅ Store files in **object storage** (AWS S3); keep only the URL/path in the DB.
- Lets ANY server serve the file (central shared store = stateless web tier).
- Serve via **CDN** for geographic speed. → Day 4

---

## Vocabulary to Remember
- SPOF = Single Point of Failure
- Stateless vs Stateful
- Object Storage (S3)
- Load Balancer (the tool that routes traffic across servers → Day 2)

---

## Personal Notes (gaps to fix)
- [ ] Finish thoughts: don't just DIAGNOSE ("X is broken") — PRESCRIBE ("so I'd do Y").
- [ ] Use proper NAMES for concepts (e.g. "object storage / S3", not "upload to AWS").
