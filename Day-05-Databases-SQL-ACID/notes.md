# Day 5 — Databases Deep Dive: SQL, ACID & Transactions
**Phase 1: Foundations** | Score: 7.5/10 (re-graded — original brief omitted "lost update")

---

## Relational (SQL) DB
- Tables (rows+columns), fixed **schema**. Related via keys.
  - **Primary key** — uniquely IDs a row.
  - **Foreign key** — points to another table's PK; enforces relationships.
- Query with SQL. Examples: PostgreSQL, MySQL, Oracle, SQL Server.
- Superpower: structured relationships + strong consistency. Default for money/orders/inventory.

---

## ACID ⭐ (asked constantly)
A **transaction** = group of ops treated as one all-or-nothing unit.
- **A — Atomicity**: all ops succeed or none (rollback). No half-transfer.
- **C — Consistency**: valid state → valid state; respects constraints (balance ≥ 0, money conserved).
  (NOT the same as CAP's "C" — Day 7.)
- **I — Isolation**: concurrent transactions don't step on each other (as if sequential).
- **D — Durability**: once committed, survives crash/power loss (disk / write-ahead log).

### Write-Ahead Log (WAL) — how Durability is actually achieved
- Before changing the actual data files, the DB first appends the change to an
  append-only **log on disk**, THEN applies it to the data.
- If the DB crashes mid-operation, on restart it **replays the WAL** to recover any
  committed-but-not-yet-applied changes → no committed data is ever lost.
- Append-only sequential writes are fast; this is why durability doesn't kill performance.
- (Also the foundation of replication — Day 8 ships the WAL to replicas.)

### Map each letter to the failure it prevents (don't just list ACID!)
- Transfer ₹500 Alice→Bob: A=no half-move, C=no negative balance, I=concurrent transfers don't interfere, D=committed transfer survives crash.

---

## Isolation Levels (consistency ↔ performance)
### Read phenomena (bugs)
- **Dirty read** — read another txn's UNCOMMITTED change.
- **Non-repeatable read** — same row read twice → different values.
- **Phantom read** — same query twice → different rows (insert/delete).

### Levels (low→high)
| Level | Prevents |
|---|---|
| Read Uncommitted | nothing |
| Read Committed | dirty reads (common default) |
| Repeatable Read | dirty + non-repeatable |
| Serializable | all three (sequential) — safest, slowest |

### Race Condition (the umbrella concept) ⭐
- **Definition:** when the correctness of a result depends on the TIMING/INTERLEAVING of
  multiple concurrent operations. The same operations can give a right answer or a wrong
  one purely based on the order they happen to run in.
- Classic shape = **read-modify-write race**: two txns READ the same value, each MODIFIES
  based on that stale read, then WRITES back — neither saw the other's change.
  Example: both read balance ₹500 → both check "enough" → both deduct → overdraft.
- Lost update & write skew (below) are SPECIFIC TYPES of race condition in databases.
- Fix in general: serialize access to the shared data (locking, atomic ops, or
  Serializable isolation) so only one txn can act on it at a time.

### Update anomalies (types of race condition — SEPARATE from read phenomena!)
- **Lost update** — two txns read same value, both modify, one overwrites the other
  (the first write is "lost").
- **Write skew** — two txns read overlapping data, each writes based on a stale read,
  together they violate a constraint that neither violated alone (e.g. overdraft race).
- These are NOT the same as dirty/non-repeatable/phantom (those are READ phenomena).

---

## Locking
- **Pessimistic** — lock row BEFORE touching (`SELECT … FOR UPDATE`); others wait. Safe, can deadlock.
- **Optimistic** — no lock; at commit check version number; if changed → retry. Best for low-contention/read-heavy.

---

## ⭐ Overdraft Race (KEY correction)
- Alice (₹500) sends ₹500 to Bob AND ₹500 to Carol simultaneously.
- Both txns read ₹500, both see "enough", both deduct → overdraft.
- ❌ NOT a "dirty read" (that = reading uncommitted data).
- ✅ Correct name: **race condition / lost update / write skew** (both read same COMMITTED value, both act, constraint violated).
- Prevent: (1) **pessimistic lock** `SELECT … FOR UPDATE` on Alice's row → 2nd waits, re-reads ₹0, fails.
           (2) **optimistic lock** version check + retry. (3) Serializable isolation (heavyweight).

## Serializable everywhere = bad default
- Forces near-sequential execution → heavy locking, contention, deadlocks, retries → throughput collapses.
- Use Serializable only on critical paths (money); **Read Committed** for normal reads. Match level to risk.

---

## ⚠️ PENDING (intentionally open — follow-up to attempt first)
- **Deadlock** (referenced under Locking): Alice→Bob locks Alice then needs Bob;
  Bob→Alice locks Bob then needs Alice → each waits forever for the other.
  Full definition + prevention to be filled in AFTER I attempt the follow-up question.

---

## Personal Notes (recurring gaps)
- [ ] NAME the bug correctly: "lost update / race condition" ≠ "dirty read". (Cost a full category.)
- [ ] Include ALL of ACID — don't skip C (Consistency) in money examples.
- [x] Mapping ACID letters → specific failures was strong ✅.
- TODO: read DDIA Ch.7 (Transactions) for depth.
