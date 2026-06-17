# Day 5 — Databases Deep Dive: SQL, ACID & Transactions
**Phase 1: Foundations** | Score: 7.5/10 (re-graded — original brief omitted "lost update")

> **One-sentence idea:** A relational database stores data in linked tables and protects it with
> **ACID transactions** — a way to group several changes into one all-or-nothing unit so that even when
> the machine crashes mid-operation or two users act at the exact same instant, the data never ends up
> in a broken, half-finished, or contradictory state. Picture a **bank ledger** where a transfer is only
> "real" once BOTH the debit and the credit are written — never just one.

---

## Relational (SQL) DB — the starting point

A **relational database** stores data in **tables**: grids of rows (records) and columns (fields), like a
spreadsheet. Each table has a fixed **schema** — a contract that says exactly which columns exist and what
type each one holds (e.g. `name` is text, `balance` is a number). "Fixed" means you can't just toss in a
random extra field on one row; every row obeys the same shape. That rigidity is the point — it guarantees
the data is uniform and predictable.

Tables connect to each other through **keys**:
- **Primary key (PK)** — a column whose value uniquely identifies a row, like an account number. No two rows
  share the same PK, so it's the reliable handle for "this exact record."
- **Foreign key (FK)** — a column in one table that points to another table's primary key. Example: an
  `orders` table has a `customer_id` foreign key pointing at the `customers` table's PK. This is what
  *enforces relationships* — the database can refuse an order that references a customer who doesn't exist.

You query the data with **SQL** (Structured Query Language). Real examples: PostgreSQL, MySQL, Oracle,
SQL Server.

⭐ **Why reach for a relational DB:** it gives you structured relationships *plus* strong consistency
guarantees. That's why it's the default choice for anything involving money, orders, or inventory — places
where a wrong number is a real-world disaster.

---

## ACID ⭐ (asked constantly)

First, the key word: a **transaction** is a group of operations the database treats as **one
all-or-nothing unit**. The textbook example is a bank transfer — "subtract ₹500 from Alice" and "add ₹500
to Bob" are two operations, but they must succeed or fail *together*. ACID is the set of four guarantees a
transaction gives you.

- **A — Atomicity** — all operations in the transaction succeed, or none do (the DB performs a **rollback**,
  undoing everything). Intuition: there is no such thing as "half a transfer" — you never lose ₹500 from
  Alice without Bob receiving it.
- **C — Consistency** — the transaction moves the database from one *valid* state to another valid state,
  respecting all the rules you defined (a balance can't go below 0, total money is conserved). If a
  transaction would break a rule, it's rejected. ⚠️ This is **NOT** the same "C" as in CAP's consistency
  (that's Day 7 — there it means "all replicas show the same value"). Same letter, different idea.
- **I — Isolation** — concurrent transactions don't trip over each other; each runs *as if* it were the
  only one, even when many run at the same time. Intuition: two transfers happening at once should produce
  the same result as if they'd run one after the other.
- **D — Durability** — once a transaction is **committed** (officially saved), it survives a crash or power
  loss. The moment the DB says "done," that data is on disk for good — even if the server loses power one
  millisecond later.

### Write-Ahead Log (WAL) — how Durability is actually achieved

How does the DB promise data survives a crash? The trick is the **Write-Ahead Log (WAL)**.

The intuition: before the DB touches the real data files, it first **writes down what it's about to do** in
an append-only log on disk — *then* it goes and applies the change to the actual data. "Append-only" means
it only ever adds to the end of the log, never edits the middle. Think of a chef writing every order into a
notebook the instant it comes in, *before* starting to cook — if the kitchen catches fire, the notebook
still tells you exactly which dishes were ordered.

So if the DB crashes mid-operation, on restart it **replays the WAL**: it walks the log and re-applies any
change that was committed but hadn't yet made it into the data files. The result — no committed data is
ever lost.

⭐ **Why this doesn't kill performance:** appending to the end of a file is a *sequential* write, which is
far faster than hunting around the disk to update data files in random places. So durability is cheap
precisely because the log is append-only. (This same WAL is also the foundation of replication — Day 8
ships the WAL to replica servers to keep them in sync.)

### Map each letter to the failure it prevents (don't just list ACID!)

⭐ A common interview mistake is to recite the four words. Better: tie each letter to the concrete disaster
it stops. Transfer ₹500 Alice→Bob:
- **A** = no half-move (Alice debited, Bob never credited).
- **C** = no negative balance / money is conserved.
- **I** = two concurrent transfers don't interfere with each other.
- **D** = once committed, the transfer survives a crash.

---

## Isolation Levels (the consistency ↔ performance dial)

Isolation isn't free — the stricter you make it, the more the DB has to lock things and slow down. So
databases offer **isolation levels**: a dial trading safety for speed. To understand the dial, first meet
the bugs it's protecting against.

### Read phenomena (the read bugs)

These are the three things that can go wrong when one transaction *reads* data while another is *writing*:

- **Dirty read** — you read another transaction's change *before it's committed*. If that other transaction
  then rolls back, you acted on data that never officially existed.
- **Non-repeatable read** — you read the same row twice inside one transaction and get *different values*,
  because another transaction committed a change in between. The row "changed under your feet."
- **Phantom read** — you run the same *query* twice and get a different *set of rows*, because another
  transaction inserted or deleted rows matching your filter. New rows appear (or vanish) like phantoms.

### Levels (low→high) — SUMMARY

Each higher level prevents one more phenomenon, at the cost of more locking/slowness:

| Level | Prevents |
|---|---|
| Read Uncommitted | nothing |
| Read Committed | dirty reads (common default) |
| Repeatable Read | dirty + non-repeatable |
| Serializable | all three (behaves as if sequential) — safest, slowest |

### Race Condition (the umbrella concept) ⭐

Before the next category of bugs, define the big idea they all live under.

- **Definition:** a **race condition** is when the correctness of a result depends on the *timing /
  interleaving* of multiple concurrent operations. The exact same operations can produce a right answer or
  a wrong one purely based on the order they happen to run in — they're "racing," and who wins changes the
  outcome.
- **The classic shape = a read-modify-write race:** two transactions both READ the same value, each MODIFIES
  it based on that (now stale) read, then each WRITES the result back — and neither one saw the other's
  change. Concrete example: both read Alice's balance ₹500, both check "is there enough?", both say yes,
  both deduct → Alice overdraws.
- **Lost update** and **write skew** (next section) are SPECIFIC TYPES of race condition that happen in
  databases.
- **The general fix:** *serialize* access to the shared data — force the conflicting operations to take
  turns so only one transaction can act on it at a time. You do that with locking, atomic operations, or
  Serializable isolation.

### Update anomalies (types of race condition — SEPARATE from read phenomena!)

⚠️ Keep these mentally separate from the read phenomena above. Read phenomena are about *reading* wrong
data; these are about concurrent *writes* clobbering each other.

- **Lost update** — two transactions read the same value, both modify it, and one's write silently
  overwrites the other's. The first write is "lost" as if it never happened. Example: two people edit the
  same wiki page from a stale copy; whoever saves last wipes out the other's edit.
- **Write skew** — two transactions read overlapping data, each writes based on a stale read, and
  *together* they violate a constraint that *neither* would have violated alone. The overdraft race is the
  textbook case: each transfer alone looked fine, but run concurrently they break the "balance ≥ 0" rule.

⭐ Remember: lost update and write skew are **NOT** the same as dirty / non-repeatable / phantom reads.
Those are READ phenomena; these are WRITE anomalies. Naming them correctly is itself an exam point.

---

## Locking — the two strategies for taking turns

When you need to stop transactions from clobbering each other, you lock the data. Two philosophies:

- **Pessimistic locking** — assume a clash *will* happen, so grab the lock *before* you touch the row
  (in SQL: `SELECT … FOR UPDATE`). Anyone else who wants that row **waits** until you're done. It's safe,
  but because transactions sit around holding locks and waiting, it can lead to **deadlock** (see below).
  Analogy: you lock the bathroom door before using it; others queue.
- **Optimistic locking** — assume clashes are *rare*, so don't lock at all. Instead, each row carries a
  **version number**; at commit time you check "is the version still what I read?" If it changed, someone
  else got there first → you **retry**. Best when contention is low and reads dominate (you rarely pay the
  retry cost). Analogy: you don't lock the door, but if you discover someone slipped in, you start over.

---

## ⭐ Overdraft Race (KEY correction)

This is the worked example that ties race conditions, anomalies, and locking together — and it's where I
got the *name* wrong, so it's worth nailing.

The scenario: Alice has ₹500. She sends ₹500 to Bob AND ₹500 to Carol at the **same instant**. Both
transactions read her balance as ₹500, both conclude "she has enough," both deduct ₹500 → Alice ends up
overdrawn at −₹500. The constraint "balance ≥ 0" is violated even though each transfer looked valid alone.

- ❌ This is **NOT** a "dirty read." A dirty read means reading *uncommitted* data — but here both
  transactions read the same fully **committed** ₹500. Calling it a dirty read is the classic mislabel.
- ✅ The correct name is a **race condition / lost update / write skew** — both read the same committed
  value, both act on it, and the constraint is violated.
- **How to prevent it:**
  1. **Pessimistic lock** — `SELECT … FOR UPDATE` on Alice's row. The first transaction locks it; the
     second must wait, then re-reads the now-₹0 balance and correctly fails.
  2. **Optimistic lock** — version check at commit + retry the loser.
  3. **Serializable isolation** — forces the two to behave as if sequential (heavyweight; see next).

## Serializable everywhere = bad default

It's tempting to just crank every transaction to Serializable and stop worrying. Don't.

Serializable forces near-sequential execution, which means heavy locking → lots of contention (transactions
queuing on each other) → more deadlocks → more retries → **throughput collapses**. You bought perfect
safety by making the database slow for everyone.

⭐ The right move is to **match the isolation level to the risk**: use Serializable only on the critical
paths where a wrong answer is catastrophic (money, inventory), and use a cheaper level like **Read
Committed** for ordinary reads (showing a profile, listing comments) where a tiny staleness is harmless.

---

## ⭐ Deadlock (follow-up — RESOLVED)

- **Definition:** a **deadlock** is when two (or more) transactions each hold a lock the other needs, so
  each waits forever for the other to release → neither can ever proceed. It's a *cyclic* wait-for
  dependency — a circle of "I'm waiting on you, who's waiting on me."
- **Classic shape (two mutual transfers at once):**
  - Txn 1: transfer Alice→Bob → locks **Alice's** row, then asks for **Bob's**.
  - Txn 2: transfer Bob→Alice → locks **Bob's** row, then asks for **Alice's**.
  - Now Txn 1 is stuck waiting on Txn 2's lock of Bob, and Txn 2 is stuck waiting on Txn 1's lock of Alice
    → a closed cycle → deadlock. Like two people in a doorway, each refusing to step back until the other
    does.
- **The 4 conditions (ALL must hold — the Coffman conditions):**
  1. **Mutual exclusion** — a lock is held by only one transaction at a time.
  2. **Hold-and-wait** — a transaction holds one lock while requesting another.
  3. **No preemption** — a lock can't be forcibly taken away; only the holder can release it.
  4. **Circular wait** — there's a cycle of transactions each waiting on the next.
  ⭐ Break **any one** of these four and deadlock becomes impossible — that's the lever every prevention
  trick below pulls.
- **How the DB handles it = DETECT + recover** (the default in Postgres / MySQL / InnoDB):
  - The DB maintains a **wait-for graph** (who is waiting on whom). If it spots a cycle, it picks a
    **victim** transaction — usually the cheaper or younger one — and **rolls it back** so the other can
    proceed. The victim then retries.
  - Consequence for you: the application must be ready to **catch a "deadlock detected" error and retry**
    the transaction. The DB unblocks itself, but your code has to redo the killed work.
- **How to PREVENT it (the design side):**
  1. ⭐ **Consistent lock ordering** — always acquire rows in the same global order (e.g. always lock the
     *lower* account_id first). Then both transfers grab Alice before Bob, so no cycle can form. (This
     breaks **circular wait**.)
  2. **Keep transactions short** — grab all needed locks fast and commit quickly, shrinking the window in
     which a clash can happen.
  3. **Lock everything up front** in one step instead of incrementally. (This breaks **hold-and-wait** — you
     never hold one lock while reaching for another.)
  4. **Lock timeouts** — give up and retry rather than waiting forever.
  5. **Optimistic locking** — avoid holding locks at all: check the version at commit time and retry on
     conflict.
- ⚠️ **Don't confuse deadlock with its cousins:** a **livelock** is when transactions keep retrying but
  still block each other (they're active, not frozen, but make no progress); plain **lock contention** is
  when one transaction just waits a bit and then proceeds normally. A deadlock is specifically *mutual,
  cyclic, and permanent* — no one ever gets out without intervention.
- **Interview one-liner:** "Deadlock = cyclic wait on locks; the DB detects the cycle and kills a victim,
  so I make it rare with **consistent lock ordering** + short transactions, and make the app **retry**."

---

## Personal Notes (recurring gaps)
- [ ] NAME the bug correctly: "lost update / race condition" ≠ "dirty read". (Cost a full category.)
- [ ] Include ALL of ACID — don't skip C (Consistency) in money examples.
- [x] Mapping ACID letters → specific failures was strong ✅.
- TODO: read DDIA Ch.7 (Transactions) for depth.
