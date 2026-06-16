# Day 7 — CAP Theorem + PACELC
**Phase 1: Foundations** | Score: 7.5/10

---

## Why CAP exists
- Data on >1 machine (replicas for fault tolerance) → nodes can disagree + network can fail.
- CAP describes the unavoidable trade-off when that happens.

## The 3 letters
- **C — Consistency**: every read gets the MOST RECENT write (all nodes show same data now).
  ⚠️ NOT the same as ACID's C. CAP-C = "all replicas agree on latest value RIGHT NOW".
- **A — Availability**: every request gets a non-error response (maybe stale).
- **P — Partition tolerance**: keeps working even when network between nodes fails.

## The theorem (state precisely)
> During a network partition (P), you must choose between C and A. Can't have both DURING a partition.
- ⭐ KEY NUANCE: P is MANDATORY in real distributed systems (networks fail). So you don't
  "pick 2 of 3" — the ONLY real choice is **C vs A, and only during a partition**.
  No partition → you get BOTH C and A. CAP describes behavior DURING A FAILURE, not always.

## During a partition (Node A can't reach Node B; write hits A)
- **CP (choose Consistency):** A refuses/blocks the request → consistent but UNAVAILABLE.
  "Better an error than a wrong answer."
- **AP (choose Availability):** A accepts + serves stale → available but INCONSISTENT
  (reconcile later / eventual consistency). "Better a stale answer than no answer."

## The 3 combinations
- **CP** — sacrifices availability during partition. Ex: HBase, MongoDB (default), Zookeeper, etcd, clustered RDBMS.
- **AP** — sacrifices consistency during partition. Ex: Cassandra, DynamoDB, Riak, CouchDB.
- **CA** — NOT achievable in a real distributed system (can't avoid partitions).
  A single node is "CA" only trivially (it isn't distributed).

## Mapping to decisions (by CONSEQUENCE)
- Banking / payments / inventory / seat-booking → **CP** (wrong data catastrophic).
- Social feeds / likes / DNS / cart-adds → **AP** (stale is fine; uptime matters more).

---

## PACELC (the upgrade — impresses)
> If Partition (P): choose A vs C. Else (E, normal operation): choose Latency (L) vs Consistency (C).
- KEY: even with NO partition there's a latency↔consistency trade-off.
  Strong consistency = wait for multiple replicas to confirm a write → slower (latency).
  Low latency = return before all replicas confirm → risk stale reads.
- ⭐ ALWAYS answer BOTH halves of a PACELC class.
- Classes:
  - **Cassandra / DynamoDB = PA/EL** — partition→Availability; normal→Latency (speed+uptime first).
  - **MongoDB = PA/EC** (tunable) — normal→Consistency.
  - **RDBMS / Google Spanner = PC/EC** — consistency always (cost: availability in partition, latency normally).

---

## Myths to NEVER repeat
- ❌ "Pick any 2 of 3 freely" — P is mandatory; real choice = C vs A during partition only.
- ❌ "My distributed system is CA" — not real.
- ❌ "CAP applies all the time" — only during a partition (that's why PACELC exists).
- ❌ Confusing CAP-C with ACID-C.

---

## ⭐ The "CA single server" trap (Q5 — answer, don't deflect)
- Calling single-server Postgres "CA" is wrong because:
  1. CAP only applies to DISTRIBUTED systems — one node has no partitions/replicas, so "CA" is meaningless.
  2. A single server is the OPPOSITE of available — it's a SPOF (Day 1); if it dies, availability = 0.
- True availability REQUIRES redundancy → requires being distributed → forces the real C-vs-A choice.
- You can't escape CAP by avoiding distribution; you just trade it for a SPOF.

---

## ⭐ Offline ATM = AP banking (follow-up — RESOLVED)
- WHY AP: a working ATM has real business value (happy customers); dead ATMs region-wide
  during a network blip is worse than a rare bounded loss. CAP choices FOLLOW MONEY, not dogma.
  (Same bank is CP for online transfers but AP at the ATM — decided per-feature.)
- HOW made safe = **bound the blast radius**: apply a LOCAL offline withdrawal limit (e.g. ₹10k)
  with NO coordination (can't lock across nodes during a partition — that's the whole point).
  - NB: it's an "offline limit", NOT a "lock". A global lock needs the network that's broken.
- Reconcile when online: offline txns queued/logged locally → replayed centrally → converge
  (eventual consistency). If overdrawn → overdraft fee / recover AFTER the fact.
- PRINCIPLE: make AP "safe enough" by bounding the worst case + reconciling after heal
  (DETECT + RECOVER instead of PREVENT). Prevention was too expensive (availability cost).

---

## Personal Notes (recurring gaps)
- [ ] Answer EVERY part of a multi-part question (dropped "PA" half of PACELC in Q4).
- [ ] Don't ask a clarifying question whose answer is already in the prompt (Q5) — answer it.
- [ ] Watch wording precision ("can't have both DURING a partition", not "can have both").
- [x] CP/AP classification by consequence = 3/3, strong ✅. ATM double-spend risk spotted ✅.
