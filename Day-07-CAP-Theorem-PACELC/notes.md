# Day 7 — CAP Theorem + PACELC
**Phase 1: Foundations** | Score: 7.5/10

> **One-sentence idea:** Once your data lives on more than one machine, the network connecting them *will*
> sometimes break — and in that moment you can't keep every copy in agreement AND answer every request, so
> you must pick one. Think of **two clerks in two buildings sharing one ledger by phone**: when the phone
> line drops, each clerk either *refuses to serve* (so the books never disagree) or *keeps serving from
> memory* (so customers are happy, but the two ledgers drift apart). CAP is just naming that forced choice.

---

## Why CAP exists
The whole reason this theorem exists is that we put data on more than one machine in the first place. We do
that for fault tolerance — keeping **replicas** (extra copies of the same data on other servers) so that one
dead machine doesn't kill the app.

But the moment you have copies on separate machines, two new problems appear: the nodes can **disagree**
(one got a write the other hasn't seen yet), and the **network between them can fail** (cables cut, switches
die, packets lost). CAP is simply the name for the unavoidable trade-off you face when that network failure
happens. It's not abstract math for its own sake — it's describing what physically *must* go wrong.

## The 3 letters
CAP stands for three properties you'd love to have all at once. Define each one precisely, because the exact
wording is where people trip:

- **C — Consistency:** every read returns the **most recent write** — i.e. all nodes show the same data
  *right now*. If you just wrote "balance = ₹500", any read from any node immediately returns ₹500, never an
  older ₹700.
  - ⚠️ **This is NOT ACID's "C".** ACID-C (from databases, Day 5) means "a transaction never violates a rule
    like 'debits must equal credits'." CAP-C means something different: **"all replicas agree on the latest
    value right now."** Same letter, different concept — don't mix them up.
- **A — Availability:** every request gets a **non-error response** — the system answers, even if the answer
  might be a little **stale** (out of date). "I always reply" — not "I always reply *correctly*."
- **P — Partition tolerance:** the system **keeps working even when the network between nodes fails**. A
  "partition" is when nodes can't talk to each other — the cluster is split into islands that can't sync.
  - ⚠️ **Two unrelated uses of the word "partition" — don't confuse them:**
    - **Network partition (CAP sense, used here)** = a *failure event* — the link between nodes breaks.
    - **Data partition = sharding** = a *design choice* — splitting the dataset into disjoint pieces, each
      on a different node (users A–M on node 1, N–Z on node 2). Nothing to do with CAP.
  - And don't confuse either of those with **replicas**: a replica is the *same* data copied to another
    node (for redundancy); a shard is *different* data on another node (for scale). Real systems usually
    do both — shard the data, then replicate each shard.

## The theorem (state it precisely)
> **During a network partition (P), you must choose between C and A. You cannot have both DURING a partition.**

Why? Picture node A and node B, normally syncing. The link between them dies (partition). A write arrives at
A. A has two options: tell B before confirming (impossible — B is unreachable, so A would have to **block**
and refuse → loses *availability*), or confirm without B (now A and B disagree → loses *consistency*).
There's no third door while the link is down.

- ⭐ **KEY NUANCE — you do NOT "pick 2 of 3".** In any real distributed system, **P is mandatory**: networks
  fail whether you like it or not, so you can't opt out of partition tolerance. That means the real choice
  isn't "which two of C/A/P" — it's narrower and more specific: **C vs A, and only during a partition.**
  - When there's **no partition** (the normal, healthy state — which is *most* of the time), you happily get
    **both C and A**. CAP only describes behavior **during a failure**, not all the time. This is the single
    most misunderstood point about the theorem.

## During a partition — what each choice actually does
Concrete setup: Node A can't reach Node B (partition), and a write lands on A. A must decide:

- **CP (choose Consistency):** A **refuses or blocks** the request rather than risk being out of sync. Result:
  the data stays correct everywhere, but A is **unavailable** — the user gets an error or a hang.
  - Motto: *"Better an error than a wrong answer."*
- **AP (choose Availability):** A **accepts the write (or serves a read) anyway**, knowing its data may be
  **stale** relative to B. Result: A keeps answering, but the two sides are now **inconsistent**; you
  **reconcile later** once the link heals (this is where **eventual consistency** — copies agree
  *eventually*, not instantly — comes from).
  - Motto: *"Better a stale answer than no answer."*

## The 3 combinations (CP / AP / CA)
- **CP** — sacrifices **availability** during a partition to stay consistent.
  - Examples: HBase, Zookeeper, etcd, and **distributed/clustered** relational databases (RDBMS).
    - ⚠️ **On "RDBMS":** a *single-node* RDBMS isn't distributed, has no network between nodes, and so has
      **no partition** — it gets no meaningful CAP class at all (see the "CA single server" trap below). It's
      the **distributed/clustered** RDBMS setup (with replicas across nodes) that earns a CAP class.
    - **MongoDB is NOT a flat CP entry** — its behavior is tunable by write concern; see its PACELC class
      below for the precise treatment.
- **AP** — sacrifices **consistency** during a partition to stay available.
  - Examples: Cassandra, DynamoDB, Riak, CouchDB.
- **CA** — **not achievable in a real distributed system**, because you can't magically prevent partitions.
  A single, non-distributed node can be called "CA" only trivially — and that's cheating, because it isn't
  distributed at all (more on why this is a trap below).

## Mapping to decisions — choose by CONSEQUENCE
The right letter isn't a matter of taste; it follows from **how bad wrong data is** for that feature:

- **CP** when wrong/stale data is catastrophic: **banking, payments, inventory counts, seat-booking.** Selling
  the same airplane seat twice or showing a wrong balance is unacceptable, so you'd rather refuse than err.
- **AP** when staleness is harmless but downtime hurts: **social feeds, like counts, DNS, adding to a cart.**
  A like count that's briefly off, or a feed half a second behind, is fine; being *down* loses users.

---

## PACELC — the upgrade that impresses
CAP only talks about the partition case. **PACELC is *CAP + an extra clause*** — it covers BOTH the
partition case AND normal operation (where the system actually spends most of its life).

> **If Partition (P): choose A vs C. Else (E, normal operation): choose Latency (L) vs Consistency (C).**

⭐ **Read it as TWO clauses that BOTH always apply** — not as "PACELC kicks in only when there's no
partition." The "P..." half restates CAP for the failure case; the "E..." half is the NEW contribution
covering normal operation. A label like **PA/EL** is making two statements at once:
- **PA:** when the network breaks → favor Availability over Consistency.
- **EL:** when the network is fine → favor Latency over Consistency.

Both halves describe the same system, just in two different situations it will spend its life in.

### The "Else" clause — what trade-off it adds
The insight CAP missed: **even with NO partition, there's still a trade-off** — between **latency** (speed)
and consistency. Here's why, concretely:
- **Strong consistency** means a write must **wait for multiple replicas to confirm** it before returning →
  the user waits longer → higher **latency**.
- **Low latency** means you **return before all replicas confirm** → faster, but a read might catch a replica
  that hasn't received the write yet → **stale read**.

So you're trading speed against correctness *all the time*, not just during failures. That's why PACELC
exists: CAP had nothing to say about the 99% of the time the network is healthy, even though the trade-off
is still happening then.

- ⭐ **ALWAYS answer BOTH halves of a PACELC class** — the "P..." part and the "E..." part. Dropping one half
  is a classic mistake (it cost me points in Q4).
- Classes to know:
  - **Cassandra / DynamoDB = PA/EL** — on **P**artition choose **A**vailability; **E**lse choose **L**atency.
    Speed and uptime first, both during failures and normal operation.
  - **MongoDB = PC/EC** (tunable) — on **P**artition it chooses **C**onsistency: the primary steps down and
    the minority side **refuses writes** (consistency over availability); **E**lse it also favors
    **C**onsistency (reads default to the primary). ⭐ **Nuance — why some sources say PA/EC:** it depends on
    **write concern.** With `w:majority` a write must be acknowledged by a majority before returning →
    PC-leaning (won't confirm on the minority side). With `w:1` (ack from the primary only) it leans PA —
    faster, but a failover can lose that un-replicated write. So the "right" letters follow the config you run.
  - **Distributed RDBMS / Google Spanner = PC/EC** — **C**onsistency always. The cost: it gives up
    availability during a partition, and accepts higher latency during normal operation, all to never be wrong.

---

## Myths to NEVER repeat
Each of these is a trap an interviewer is hoping you'll fall into:

- ❌ **"Pick any 2 of 3 freely."** No — P is mandatory, so the only real choice is **C vs A, and only during
  a partition.**
- ❌ **"My distributed system is CA."** Not real — you can't avoid partitions, so true CA doesn't exist in a
  distributed system.
- ❌ **"CAP applies all the time."** No — it only describes behavior **during a partition.** (That's exactly
  the gap PACELC was invented to fill.)
- ❌ **"PACELC only applies when there's no partition."** No — PACELC has TWO clauses that BOTH always
  apply. The "P..." half covers the partition case (same as CAP); the "E..." half covers normal operation.
  PACELC = CAP + Else clause, not a replacement that kicks in only when the network is fine.
- ❌ **Confusing CAP-C with ACID-C.** CAP-C = "all replicas agree on the latest value now"; ACID-C =
  "transactions don't break data rules." Different things.
- ❌ **Confusing "partition" (network failure, CAP sense) with "partition" (sharding, data-split sense).**
  Same word, unrelated ideas. CAP only ever means the failure sense.

---

## ⭐ The "CA single server" trap (Q5 — answer it, don't deflect)
Someone calls a single-server Postgres "CA" (consistent + available, no partitions). It's tempting to nod —
but it's **wrong**, for two reasons:

1. **CAP only applies to DISTRIBUTED systems.** One node has no replicas and no network between nodes, so
   there are no partitions to tolerate — calling it "CA" is meaningless, like grading a swimmer on their
   cycling.
2. **A single server is the OPPOSITE of available.** It's a **SPOF** (single point of failure, Day 1): if
   that one machine dies, availability drops to **zero.** Calling it "highly available" is backwards.

The deeper lesson: **true availability REQUIRES redundancy → which requires being distributed → which forces
the real C-vs-A choice.** You can't escape CAP by refusing to distribute; you just trade the C-vs-A dilemma
for a SPOF, which is worse.

---

## ⭐ Offline ATM = AP banking (follow-up — RESOLVED)
A puzzle: banking is supposed to be CP, yet an offline ATM that keeps dispensing cash during a network outage
is behaving as **AP.** Why is that the *right* call, and how is it made safe?

**WHY it's AP:** a working ATM has real business value — happy customers, money moving. Compare the
alternatives: every ATM in a region going **dead** during a brief network blip is *worse* for the bank than a
rare, **bounded** loss from someone overdrawing. ⭐ **CAP choices follow the money, not dogma.** The very same
bank is **CP for online transfers** (where a double-spend is catastrophic) but **AP at the ATM** — the choice
is made **per-feature**, not once for the whole company.

**HOW it's made safe — bound the blast radius:** during the outage, the ATM enforces a **local offline
withdrawal limit** (e.g. ₹10,000) with **no coordination** with other nodes. Coordination is impossible
anyway — you can't lock an account across machines when the network linking them is down (that's the whole
definition of a partition).
- NB on wording: it's an **"offline limit", NOT a "lock."** A global lock would need the very network that's
  broken; a local cap needs nothing but the ATM itself.

**Reconcile when back online:** the offline transactions are **queued/logged locally**, then **replayed
centrally** when connectivity returns, so all copies **converge** (eventual consistency). If an account ends
up overdrawn, the bank charges an **overdraft fee** or recovers the money **after the fact.**

**The principle:** make an AP design "safe enough" by **bounding the worst case + reconciling after the
network heals** — i.e. **DETECT + RECOVER instead of PREVENT.** Prevention (refusing all withdrawals) was
simply too expensive here, because its cost is availability — the one thing the ATM most needs.

---

## ⭐ Interview-completeness additions (audit pass)
CAP/PACELC give you the *labels*; interviewers push one level deeper into **what "consistency" actually
means**, **how you tune it**, and **what machinery makes CP possible**. Here's the missing layer.

### 1. Consistency models — the spectrum (the biggest gap)
"Consistent" isn't one thing — it's a dial from strict to loose. Learn the three anchor points:
- **Strong (linearizable):** every read sees the **most recent write**, as if there were only **one single
  copy** of the data. The system behaves like a single machine even though it's many. This is exactly CAP's
  "C" — and it's **expensive**, because every operation needs **coordination** across replicas before it can
  return. Example: after you transfer money, *every* branch instantly shows the new balance.
- **Causal:** operations that have a **cause-and-effect relationship** are seen in the **right order
  everywhere**; operations that are *unrelated* can be seen in any order. The classic guarantee: you never
  **see a reply before the question it answers**. Cheaper than strong (only causally-linked ops need
  ordering), and usually "good enough" for human-facing apps.
- **Eventual:** if writes **stop**, all replicas will **eventually converge** to the same value. That's *all*
  it promises — it says **nothing about when**, and nothing about the ordering you'll see in the meantime.
  You might read new-then-old-then-new. Example: a like count that settles after a few seconds.

Intuition: strong = "always current, always coordinated, always slower"; eventual = "fast and always up, but
temporarily wrong"; causal sits in between and preserves the orderings humans actually notice.

### 2. The four client-centric (session) guarantees — name them
These are weaker, *per-session* promises an AP system can still give you so it doesn't feel broken. Know all
four by name (interviewers love the checklist):
- **Read-your-own-writes:** after **you** write, **your** own reads see it. You post a comment, reload the
  page, and it's there — even if other users don't see it yet.
- **Monotonic reads:** you never see an **older** value on a **later** read. A comment you already saw
  doesn't **vanish** on the next refresh (no going backwards in time).
- **Monotonic writes:** your writes are applied in the **order you issued** them (write 1 lands before write 2).
- **Writes-follow-reads:** if you **read X** and then **write Y**, anyone who sees **Y** must also see **X** —
  a **reply** implies the **original** it responds to is already visible.

⭐ **Cross-ref:** these four are exactly the anomalies **replication lag** causes (Day 8), viewed from the
consistency-model side. Same phenomenon, two vocabularies: Day 8 says "lag caused the comment to vanish";
here we say "monotonic reads was violated."

### 3. Linearizability vs serializability (a classic trap)
Same-sounding words, **orthogonal** guarantees — mixing them up is an instant tell:
- **Linearizability** = CAP's **"C"** = a **RECENCY** guarantee on a **SINGLE object**: single-key operations
  appear to happen in **real-time order** (once a write completes, all later reads see it). Says nothing about
  transactions.
- **Serializability** = ACID's **"I"** (isolation, Day 5) = a **multi-object TRANSACTION** guarantee: the
  result is equivalent to running the transactions in **some serial order**. Says **nothing about real-time
  recency** — that "serial order" needn't match wall-clock order.
- **Strict serializability** = **both** at once (real-time order **and** transaction isolation). Google
  **Spanner** is the poster child.
- **Tell:** **CAP-C = single-object recency; ACID-I = multi-object isolation.**

### 4. Tunable consistency — "AP" is a default, not a ceiling
An AP database isn't *stuck* being weak — you can **dial consistency per request**:
- **Cassandra** consistency levels: **ONE / QUORUM / ALL.** If you set reads and writes so that **W + R > N**
  (replicas written + replicas read exceeds total replicas), the read and write sets **overlap**, so a read
  is guaranteed to see the latest write → **strong consistency on demand**, at the cost of higher latency.
- **DynamoDB:** **eventually-consistent read** (default, cheap) vs **strongly-consistent read** (a flag, roughly
  **2x the cost**).
- Takeaway: calling something an **"AP database"** describes its **default**, not a hard limit — you choose
  the guarantee per operation and pay for it in latency/cost.

### 5. Conflict resolution — how divergent copies merge (cross-ref Day 8)
An AP system lets both sides accept writes during a partition, so when the partition **heals** the copies
have **diverged** and must be reconciled. The toolkit:
- **LWW (last-write-wins):** keep the write with the newest **timestamp.** Simple, but **lossy** (the losing
  write is silently dropped) and vulnerable to **clock skew** across machines.
- **Vector clocks / version vectors:** metadata that lets you **detect** whether two writes were **concurrent**
  (true conflict) or one **causally followed** the other (safe to order) — instead of guessing by timestamp.
- **CRDTs (conflict-free replicated data types):** data structures designed to **merge without losing data**
  (e.g. a counter that sums both sides). Best when applicable.
- Plus the repair machinery: **read-repair** (fix stale replicas on read), **anti-entropy** (background
  sync), **hinted handoff** (a live node holds writes for a downed peer and delivers them later).
- ⭐ **Full treatment is Day 8** — here just know the *names* and that conflict resolution is the price of AP.

### 6. Consensus (Paxos / Raft) — what actually makes CP work
CP systems (**Zookeeper, etcd, Spanner**) stay consistent *even during partitions* because they run a
**consensus protocol** — **Raft** or **Paxos**. The rule: a write **commits only when a majority (a quorum)
of nodes agrees** on it. Now watch what that does during a partition:
- The **majority** side can still reach a quorum → it keeps committing writes, correctly.
- The **minority** side **cannot** reach a majority → so it **refuses to serve.**

⭐ That refusal **IS** the "sacrifice availability" of CP — it's not a bug, it's the design. By requiring a
majority, the system makes it **impossible** for two partitioned halves to both accept conflicting writes
(only one side can ever hold the majority). Consensus is the machinery that turns "we chose C" into a working
guarantee.

---

## Personal Notes (recurring gaps)
- [ ] Answer EVERY part of a multi-part question (dropped "PA" half of PACELC in Q4).
- [ ] Don't ask a clarifying question whose answer is already in the prompt (Q5) — answer it.
- [ ] Watch wording precision ("can't have both DURING a partition", not "can have both").
- [ ] PACELC = CAP **+** Else clause — both halves always apply (not just the "Else" half).
- [x] CP/AP classification by consequence = 3/3, strong ✅. ATM double-spend risk spotted ✅.