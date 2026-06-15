# 🎯 System Design Interview Prep — 60-Day Program
Author: **Raghav Gupta** | Mentor: Alex | Started: 2026-06-11 | Level at start: Absolute Beginner

> My own quick-recap notes, written by me, day by day. Each folder = one day, point-to-point only.

---

## 📊 Progress Tracker

| Day | Topic | Phase | Status | Score |
|-----|-------|-------|--------|-------|
| 1 | Scalability: Vertical vs Horizontal | 1 — Foundations | ✅ Done | 7/10 |
| 2 | Load Balancers (L4/L7, algorithms, sticky sessions) | 1 — Foundations | ✅ Done | 8/10 |
| 3 | Caching (cache-aside, write-through/back, eviction, Redis vs Memcached) | 1 | ✅ Done | 6.5 → 8/10 (retry) |
| 4 | CDN & DNS (Anycast, TTL) | 1 | ✅ Done | 8/10 |
| 5 | Databases Deep Dive — SQL, ACID, Transactions | 1 | ✅ Done | 7.5/10 (re-graded) |
| 6 | NoSQL Deep Dive — types & when to use | 1 | ⬜ Next | — |
| 7 | CAP Theorem + PACELC | 1 | ⬜ | — |
| 8 | Replication (master-slave, multi-master, sync/async) | 1 | ⬜ | — |
| 9 | Sharding & Partitioning (hotspot problem) | 1 | ⬜ | — |
| 10 | Indexing (B-Tree, LSM, inverted, composite) | 1 | ⬜ | — |
| 11 | Message Queues & Event Streaming (Kafka, RabbitMQ, SQS) | 1 | ⬜ | — |
| 12 | APIs (REST/GraphQL/gRPC, gateway, idempotency) | 1 | ⬜ | — |
| 13 | Proxies & Reverse Proxies (Nginx, service mesh) | 1 | ⬜ | — |
| 14 | Rate Limiting (token/leaky bucket, sliding window) | 1 | ⬜ | — |
| 15 | Back-of-Envelope Estimation | 1 | ⬜ | — |

*(Days 16–60 added as we progress.)*

---

## Phases
- **Phase 1 (Days 1–15):** Foundations
- **Phase 2 (Days 16–30):** Design Patterns & Trade-offs
- **Phase 3 (Days 31–52):** Real System Designs
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
- Finish thoughts: diagnose → prescribe. *(improving — Day 2 prescriptions were solid)*
- Use precise vocabulary / proper names ("chunks" not "packets", "JWT" not "IndexedDB").
- ⭐ SIGNATURE LESSON (3 days running): **state lives in a SHARED store, never on the server.**

## 📈 Stats
- Days done: 5/60 | Avg score: **7.8/10** | Trend: 7 → 8 → 8(retry) → 8 → 7.5
- Day 5 re-graded 6.5→7.5: the brief omitted "lost update", so the misname wasn't fully on me.
- 🎯 Concept mastery solid; precision of vocabulary is the gap between 7.5 and 9.
- 📌 INTERVIEW SKILL: when pushed past your prep, reason aloud + hedge the name
  ("a concurrency anomaly where both txns read stale state") — that still scores.