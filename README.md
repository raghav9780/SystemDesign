# 🎯 System Design Interview Prep — 60-Day Program
Author: **Raghav Gupta** | Mentor: Alex | Started: 2026-06-11 | Level at start: Absolute Beginner

> My own quick-recap notes, written by me, day by day. Each folder = one day, point-to-point only.

> 📚 **Curriculum source-of-truth:** [`CURRICULUM-REFERENCE.md`](CURRICULUM-REFERENCE.md) — an 18-cluster
> audit of what real 2025-26 interviews test, used to make every brief complete. Mentor rules live in
> `.claude/skills/system-design-mentor/`.

---

## 📊 Progress Tracker

| Day | Topic | Phase | Status | Score |
|-----|-------|-------|--------|-------|
| 1 | Scalability: Vertical vs Horizontal | 1 — Foundations | ✅ Done | 7/10 |
| 2 | Load Balancers (L4/L7, algorithms, sticky sessions) | 1 — Foundations | ✅ Done | 8/10 |
| 3 | Caching (cache-aside, write-through/back, eviction, Redis vs Memcached) | 1 | ✅ Done | 6.5 → 8/10 (retry) |
| 4 | CDN & DNS (Anycast, TTL) | 1 | ✅ Done | 8/10 |
| 5 | Databases Deep Dive — SQL, ACID, Transactions | 1 | ✅ Done | 7.5/10 (re-graded) |
| 6 | NoSQL Deep Dive — types & when to use | 1 | ✅ Done | 8/10 |
| 7 | CAP Theorem + PACELC | 1 | ✅ Done | 7.5/10 |
| 8 | Replication (master-slave, multi-master, sync/async) | 1 | ✅ Done | 7/10 |
| 9 | Sharding & Partitioning (hotspot problem) | 1 | ✅ Done | 7/10 |
| 10 | Indexing (B-Tree, LSM, inverted, composite) | 1 | ✅ Done | 9.5/10 |
| 11 | Message Queues & Event Streaming (Kafka, RabbitMQ, SQS) | 1 | ✅ Done | 9.5/10 |
| 12 | APIs (REST/GraphQL/gRPC, gateway, idempotency) | 1 | ✅ Done | 9/10 |
| 13 | Proxies & Reverse Proxies (Nginx, service mesh) | 1 | ✅ Done | 9/10 |
| 14 | Rate Limiting (token/leaky bucket, sliding window, Redis+Lua) | 1 | ✅ Done | 8/10 |
| 15 | Back-of-Envelope Estimation | 1 | ⬜ Next | — |

### Phase 2 roadmap (set by the curriculum audit — Days 16–24 fill the biggest interview gaps)
| Day | Topic | Why it was added |
|-----|-------|------------------|
| 16 | **Answer Framework** + functional vs non-functional requirements | The spine of every 45-min interview — nothing taught it before |
| 17 | Distributed Primitives I — consensus, Raft, quorum, leader election, failure detection, gossip | Biggest senior-level gap |
| 18 | Distributed Primitives II — distributed locks + fencing tokens, logical clocks, idempotency keys, exactly-once | Correctness toolkit |
| 19 | Distributed Transactions — 2PC vs Saga, Outbox, CDC, dual-write | Microservices staple |
| 20 | Fan-out (push/pull/hybrid), Event Sourcing & CQRS | Central newsfeed/notification decision (before Twitter design) |
| 21 | Resilience — circuit breaker, bulkhead, retries+backoff+jitter, timeouts, graceful degradation | Cascading-failure prevention |
| 22 | Observability — 3 pillars, Golden Signals/RED/USE, SLI/SLO/SLA, error budgets, OpenTelemetry | Asked in nearly every senior interview |
| 23 | Security & Auth — authn/authz, OAuth2/OIDC, JWT, RBAC/ABAC, TLS/mTLS, password hashing, secrets, zero-trust | Glaring omission |
| 24 | Deployment & Release — blue-green, canary, rolling, feature flags, monolith vs microservices | How systems ship safely |
| 25–30 | Remaining design patterns + transition into real designs | — |

*(Days 31–60 detailed as we progress; Phase 3 real-design bank is in `CURRICULUM-REFERENCE.md` §4.)*

> ⚠️ **Phase 1 brief enrichments:** the audit found gaps to fold into Days 8–15 *before* I teach them
> (e.g. Day 8 leaderless + quorum W+R>N; Day 9 consistent hashing + Snowflake IDs; Day 10 LSM internals +
> geospatial; Day 11 delivery semantics + DLQ + Outbox; Day 14 distributed rate-limiting + Redis Lua).
> Days 1–7 already taught get retro-notes only if a gap is interview-critical. Full list: `CURRICULUM-REFERENCE.md` §3.

---

## Phases
- **Phase 1 (Days 1–15):** Foundations
- **Phase 2 (Days 16–30):** Design Patterns & Trade-offs (Days 16–24 set above)
- **Phase 3 (Days 31–52):** Real System Designs (question bank in `CURRICULUM-REFERENCE.md` §4)
- **Phase 4 (Days 53–60):** Interview Mastery

## Commands I can use
- `next day` / `Day N` — start a day
- `retry` — re-attempt after a low score
- `my progress` — see status, avg score, weak areas
- `mock interview` — timed simulation
- `prep for [company]` — retune for a target company
- `hint` — one nudge when stuck

---

## 🔁 Recurring Weak Areas (auto-tracked)
- **NAME the precise mechanism, don't describe it.** *(Day 9–12 gap — now largely SOLVED: Day 13 "subset/SSL
  termination/L7", Day 14 "token bucket / Lua script for atomicity / race condition" all landed cleanly.)*
- ⭐ **#1 GAP NOW — answer EVERY "name it / why" sub-part.** Day 14 was the sharpest example: ALL 4 lost points
  were unspoken clauses — the 8× consequence, the `Retry-After` purpose, why-lazy/why-atomic/why-expire. The
  *code* was senior-level; the *score* fell only because the "why" went unsaid. Same as Day 13 Req 4, Day 8 Q3,
  Day 9 Q5. **A "why?" or "name the specific X" is a scored box — tick every one.**
- Finish thoughts: diagnose → prescribe. *(largely solved — recent answers structured well)*
- Use precise vocabulary / proper names ("chunks" not "packets", "JWT" not "IndexedDB").
- ⭐ SIGNATURE LESSON: **state lives in a SHARED store, never on the server.**

## 🔍 Interview-completeness audit (Days 1–14) — DONE
A 10-agent audit + 13-agent fix pass reviewed every Day 1–14 note against `CURRICULUM-REFERENCE.md` and the
real 2025-26 interview bar. Corrected 3 factual errors (Day 5 lost-update vs write-skew; Day 7 MongoDB
PC/EC contradiction; Day 10 memtable is sorted, not hash) and appended a `⭐ Interview-completeness additions
(audit pass)` section to every day. Cores were sound; gaps were breadth (e.g. availability math, MVCC,
consistency-model spectrum, WebSocket/SSE, Kinesis, vector/time-series DBs, K8s Ingress). Each folder now
reads top-to-bottom as complete prep for that topic.

## 📈 Stats
- Days done: 14/60 | Avg score: **8.1/10** | Trend: 7 → 8 → 8(retry) → 8 → 7.5 → 8 → 7.5 → 7 → 7 → **9.5** → **9.5** → **9** → **9** → 8 (Day 14: knowledge ~9.5, score capped by unspoken "why" sub-parts — a delivery gap, not a knowledge gap)
- 🎯 Concept mastery solid; gap is INTERVIEW CRAFT + COMPLETENESS:
  (1) name the category not just the example, (2) structure answers / don't ramble,
  (3) justify by CONSEQUENCE, (4) answer EVERY part of a multi-part question,
  (5) don't ask clarifying Qs whose answer is already given — answer them.
- 📌 When pushed past prep, reason aloud + hedge the name — that still scores.
- ✅ STRENGTH: CP/AP classification by consequence (Day 7: 3/3).