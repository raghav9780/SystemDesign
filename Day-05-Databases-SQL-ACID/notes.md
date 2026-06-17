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

## ⭐ Deadlock (follow-up — RESOLVED)
- **Definition:** two (or more) transactions each hold a lock the other needs, so each waits
  forever for the other to release → neither can proceed. A *cyclic* wait-for dependency.
- **Classic shape (mutual transfer):**
  - Txn 1: transfer Alice→Bob → locks **Alice's** row, then asks for **Bob's**.
  - Txn 2: transfer Bob→Alice → locks **Bob's** row, then asks for **Alice's**.
  - Now 1 waits on 2's Bob-lock, 2 waits on 1's Alice-lock → cycle → deadlock.
- **The 4 conditions (all must hold — Coffman conditions):**
  1. **Mutual exclusion** — a lock is held by only one txn at a time.
  2. **Hold-and-wait** — a txn holds one lock while requesting another.
  3. **No preemption** — a lock can't be force-taken; only the holder releases it.
  4. **Circular wait** — a cycle of txns each waiting on the next.
  Break ANY one and deadlock is impossible.
- **How the DB handles it = DETECT + recover** (default in Postgres/MySQL/InnoDB):
  - DB maintains a **wait-for graph**; if it finds a cycle, it picks a **victim** txn and
    **rolls it back** (the cheaper/younger one) → the other proceeds; victim retries.
  - So the app must be ready to **catch a "deadlock detected" error and retry** the txn.
- **How to PREVENT it (design side):**
  1. ⭐ **Consistent lock ordering** — always acquire rows in the same global order (e.g. lock
     the lower account_id first). Both transfers grab Alice before Bob → no cycle. (Breaks circular wait.)
  2. **Keep transactions short** — acquire all needed locks fast, commit quickly → smaller window.
  3. **Lock everything up front** in one step instead of incrementally. (Breaks hold-and-wait.)
  4. **Lock timeouts** — give up + retry instead of waiting forever.
  5. **Optimistic locking** — avoid holding locks at all; check version at commit, retry on conflict.
- ⚠️ Deadlock ≠ livelock (both keep retrying and still block) and ≠ simple lock contention
  (one waits, then proceeds). Deadlock = *mutual, cyclic, permanent* wait.
- Interview one-liner: "Deadlock = cyclic wait on locks; the DB detects the cycle and kills a victim,
  so I make it rare with **consistent lock ordering** + short transactions, and make the app **retry**."

---

## Personal Notes (recurring gaps)
- [ ] NAME the bug correctly: "lost update / race condition" ≠ "dirty read". (Cost a full category.)
- [ ] Include ALL of ACID — don't skip C (Consistency) in money examples.
- [x] Mapping ACID letters → specific failures was strong ✅.
- TODO: read DDIA Ch.7 (Transactions) for depth.
