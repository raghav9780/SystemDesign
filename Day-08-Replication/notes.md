# Day 8 — Replication
**Phase 1: Foundations** | Score: 7/10

> **One-sentence idea:** Replication = keep the SAME data on more than one machine, so you survive a
> dead server and so many users can read at once. Think **photocopying one source document onto
> several desks** — the whole challenge is: *when the original is edited, how do the copies catch up,
> and what breaks in the gap before they do?*
>
> (Contrast Day 9 — **sharding splits DIFFERENT data**; replication COPIES the same data. Real systems
> do both: split data into shards, then keep replicas of each shard for safety.)

## Why replicate (3 reasons)
1. **Survival** — your one DB server dying = your app dying (the SPOF, Day 1). Copies let someone else take over.
2. **Read scaling** — one server answers only so many "show me this profile" reads/sec. 10 copies ≈ 10× read capacity.
3. **Speed for far users** — a copy in India answers an Indian user faster than reaching a US server.

The easy part is copying. The hard part is **keeping copies in sync while writes keep arriving.**

---

## A. Who is allowed to WRITE? — the 3 topologies
"Topology" = the shape of who-talks-to-whom. Three shapes:

### 1. Single-leader (master-slave / primary-replica) — ⭐ THE DEFAULT
Picture **one head chef** who alone writes into the master order-book; everyone else gets a **copy** of each order.
- ALL writes (create/update/delete) go to **one leader**. The leader streams each change to the **followers**.
- **Reads** can come from any server — leader or follower.
- Why default: simple, and never any confusion about "which version is true" — the leader IS the truth.
  Perfect for apps that read far more than they write (most apps).
- Catch: every write funnels through one leader, and if it dies you need a **failover** plan.

### 2. Multi-leader (master-master)
**Two head chefs in two cities**, each allowed to write, each copying orders to the other.
- More than one server accepts writes. Use: **multi-region writes**, or offline-capable apps (phone calendar with no signal, syncs later).
- The big catch: if both chefs edit the **same order** at once → two "truths" → a **write conflict** to resolve (Section F).

### 3. Leaderless (Dynamo-style: Cassandra, DynamoDB, Riak)
**No chef in charge.** The writer writes onto **several desks directly**; a reader checks **several desks** and picks the freshest.
- Built for maximum uptime. The "who's right?" logic moves to the client/coordinator via **quorums** (Section E).

> Beginner anchor: default to **single-leader**. Multi-leader and leaderless are tools you reach for under
> specific pressure (multi-region writes, extreme availability).

---

## B. When does the leader say "OK"? — sync vs async
A write hits the leader. It can reply "saved!" at two different moments — a real trade-off:
- **Synchronous** — leader WAITS for a follower to confirm, then says OK.
  - 👍 Safe: leader dies right after → the follower already has it, nothing lost.
  - 👎 Slow + fragile: if that follower is down/slow, the user's write **hangs**. You've tied your speed to your slowest copy.
- **Asynchronous** — leader says OK **immediately**, ships to followers in the background.
  - 👍 Fast, always responsive.
  - 👎 Leader dies in the background gap → those un-shipped writes are **GONE**.
  - ⭐ This gap is *literally where "eventual consistency" comes from* — copies are right *eventually*, not instantly.
- **Semi-synchronous** — wait for **just ONE** follower, let the rest go async. "At least one safe copy" without
  waiting on all of them. **The practical production default.**

### ⭐ MY Q1 MISTAKE: for read-heavy at scale, pick ASYNC / semi-sync — NOT full sync.
- Sync-to-all means every write blocks on the slowest of many replicas → slow + fragile.
- **The tell:** if everything were sync there'd be **no lag** → so the Q2 lag bug couldn't even happen.
  Choosing "sync everywhere" (Q1) and then fixing a "lag bug" (Q2) **contradict each other.**
  Right move: async, and hide lag only where a user observes their own action.

---

## C. The copies run BEHIND — replication lag & its 3 visible bugs
With async, followers trail the leader by ms→sec. That delay = **replication lag**. Usually invisible — but
three situations make it visible to a user, each with a standard fix. (Interviewers love these.)

**Bug 1 — "Where did my own change go?" → Read-your-own-writes (read-after-write)**
You change your profile photo (write → leader), reload instantly (read → a follower half a second behind) → it
still shows the **old** photo. You think the upload failed.
- **Fix:** for a short window after a user writes, send **THAT user's reads to the leader** (always has the latest).
  Everyone else tolerates the tiny lag.

**Bug 2 — "Time went backwards" → Monotonic reads**
You refresh twice fast. First refresh hits an up-to-date follower → shows a new comment. Second hits a more-behind
follower → the comment **vanishes**. Data moved backward in time.
- **Fix:** **pin** each user to ONE follower (e.g. hash userID → replica). Might be slightly behind, but never goes backward.

**Bug 3 — "Answer before the question" → Consistent prefix reads**
Two causally-linked writes (a question, then its answer) travel different paths and arrive out of order → reader
sees the answer appear before the question.
- **Fix:** keep causally-related writes on the **same partition** so their order is preserved.

> Pattern across all three: lag only hurts when a user observes **their own action** or a **cause→effect sequence**.
> That's exactly when you spend effort to hide it.
> ⚠️ Q2 LESSON: the photo bug is **read-your-own-writes**, NOT monotonic reads.
> read-your-writes = miss your OWN write; monotonic = data you saw goes BACKWARD. Don't swap the names.

---

## D. The leader died — failover, and the scary failure
**Failover** = promote a follower to new leader when the old one dies. Steps:
1. **Detect** it's gone (heartbeats stopped). 2. **Pick** the most caught-up follower (least lag → least data loss).
3. **Promote** it + point writes at it.

Two things go wrong:
- **Lost writes** — with async, the most caught-up follower may *still* be missing the dead leader's final writes →
  gone. (Mitigate with semi-sync: at least one other copy always had it.)
- ⭐ **SPLIT-BRAIN (the catastrophic one — I MISSED THIS in Q3):**
  The old leader wasn't actually dead — just **network-partitioned** (Day 7). You promoted a new leader. Now the old
  one returns still believing it's leader → **TWO leaders both accepting writes → diverging, corrupt data.**
  - **Prevention — two ideas together:**
    - **Quorum (majority vote):** a node can only become leader if a **majority** agrees. In a split, only the
      majority side can elect one; the lonely old leader can't get votes → can't keep ruling.
    - **Fencing token:** each new leader gets an ever-**increasing** number (#5 → #6…). The storage layer remembers
      the highest it's seen and **rejects** any write stamped with an older number → the stale old leader is denied.

---

## E. Leaderless quorums — the overlap math ⭐ (Q4 — I got 4/4)
No single "truth" node, so correctness comes from **overlap**:
- **N** = total copies of each item. **W** = copies that must confirm a WRITE. **R** = copies you READ + compare.
- ⭐ **THE RULE: `W + R > N`** → the write-set and read-set are **guaranteed to share ≥1 node** → that shared node
  has the latest write → your read can't miss it.
- **Why, concretely (N=3, W=2, R=2):** 2+2=4 > 3. There's no way to choose 2 to write and 2 to read without at
  least one node landing in BOTH groups — and that node holds the fresh value. ✅
- **Tuning the dials:** fast writes → lower W, raise R; fast reads → raise W, lower R; balanced → W=R=majority
  ((N/2)+1), i.e. N=3 → W=2,R=2.
- **Self-healing of stale copies:**
  - **Read repair** — on a read, notice a stale replica → quietly write the fresh value back to it.
  - **Anti-entropy** — a background janitor constantly compares copies and fixes differences.
  - **Hinted handoff** — target node down? another node holds the write as a "hint" and delivers it on the node's return.

---

## F. Concurrent edits clash — conflict resolution (Q5 — correct ✅)
Only an issue in multi-leader & leaderless (single-leader never has two writers).
- **LWW (Last Write Wins)** — keep the newer timestamp, drop the other. ⚠️ **silently deletes** someone's work,
  and "newer" is unreliable when server clocks disagree (clock skew).
- **Vector clocks / version vectors** — bookkeeping that tells you if two writes were *truly simultaneous*
  (a real conflict) vs one came after the other. Honest detection instead of guessing.
- **CRDTs** (Conflict-free Replicated Data Types) — data types (counters, sets) **designed to merge automatically**,
  no data loss. How Google-Docs-style collaboration works.
- **App-level merge** — hand both versions to the app, like resolving a Git merge conflict.

---

## Extras (be aware)
- **How changes physically travel:** statement-based (replay SQL — fragile: `NOW()`, randomness) /
  WAL-physical (ship the Write-Ahead Log byte-for-byte — version-coupled) / **logical row-based** ("row 7's photo
  column → X" — decoupled, enables **CDC = Change Data Capture**: stream DB changes to other systems; Day 11/19).
  Builds on Day 5's WAL.
- **Chain replication** (just know it exists): writes enter the **head**, flow down the chain, reads served from
  the **tail** → strong consistency + high throughput.

## Real-system map
- **Postgres / MySQL** → single-leader + read replicas (semi-sync option).
- **Galera / MySQL Group Replication** → multi-leader.
- **Cassandra / DynamoDB / Riak** → leaderless + quorums.

---

## ⭐ Follow-ups (RESOLVED)

### F-1. Why does adding READ replicas hurt WRITE performance? (even though writes didn't increase)
- The leader is a **single fan-out point / broadcast station**: it must personally ship every change to **every**
  follower. 5 replicas = send it 5×, 20 replicas = send it 20× → **O(N) work per write** on one machine
  (outbound bandwidth + CPU to serialize/push + one connection/thread per follower).
- So even with the *same number of writes*, the **cost per write** climbs as replicas pile up → leader saturates →
  write latency rises. (NB: this happens even with ASYNC — it's the *sending* cost, not waiting for consistency.)
- **Fixes — why you can't scale reads forever with flat replicas:**
  1. **Cascading / chained replication** — replicas replicate from *other replicas* (a tree). Leader feeds only 2–3
     nodes; they feed the next layer → leader's fan-out stays small.
  2. **Cache reads** (Day 3, Redis) → need fewer replicas.
  3. **Shard** (Day 9) → each leader owns a slice → fewer total writes + fewer replicas each.

### F-2. Multi-region semi-sync: Tokyo write blocked ~150ms waiting on a US follower — keep writes fast w/o losing durability?
- ⭐ **Region-local durability:** semi-sync only needs **ONE** follower ack — and YOU choose which. Put at least one
  replica in the **SAME region as the leader**, and make *that local replica* the one semi-sync waits on (~1ms,
  same datacenter). Cross-region replicas stay **async** (catch up in the background; user never waits on them).
- This **decouples** "is it safely persisted?" (answered locally, fast) from "is it everywhere on Earth?"
  (eventual, background). The WAN is the expensive part → never block on it.
- This is exactly how **Spanner / CockroachDB** keep global writes fast: quorum within a nearby zone, async across continents.
  - ⚠️ **Precision correction (pass 2) — the line above is overstated.** In these systems it's a *configuration
    choice*, not automatic:
    - **Region-local quorum is possible when you configure it.** Spanner **regional** configs put all 3 voting
      replicas in one region (write quorum = 2 of 3, in-region). CockroachDB's default **ZONE survival goal**
      puts a range's 3 voting replicas in its **home region** → in-region quorum, fast writes — but you can't
      survive losing that whole region.
    - **If voting replicas span regions, every write pays the cross-region round trip.** Spanner
      **multi-region** configs need a quorum that includes replicas outside the leader region, and Google's
      docs note the added write latency. CockroachDB's **REGION survival goal** (5 voters across ≥3 regions)
      states write latency rises by at least the round-trip time to the nearest other region.
    - **Only non-voting replicas lag asynchronously.** Spanner **read-only replicas** and CockroachDB
      **non-voting replicas** sit in far regions, don't count toward write quorum, and serve fast,
      possibly-stale reads.
    - So the honest rule is the same trade as F-2 itself: **local quorum = fast writes but you lose the region
      if the region dies; cross-region quorum = survive a region loss but pay the WAN on every write.**

---

## Personal Notes (recurring gaps)
- [ ] **Answer what's ASKED:** Q3 asked the *catastrophic* failure → **split-brain**; I gave only lost writes.
- [ ] **Naming precision:** read-your-own-writes ≠ monotonic reads (mislabeled Q2).
- [ ] **Internal consistency:** don't pick "sync" in Q1 then solve a "lag" bug in Q2 — they contradict.
- [x] **Quorum math W+R>N = 4/4, perfect** ✅. CRDT for conflict ✅. Fan-out + region-local durability follow-ups ✅ (senior-level).

---

## ⭐ Interview-completeness additions (audit pass)

### 1. The other two session guarantees (rounding out Day 7)
We already covered **read-your-writes**, **monotonic reads**, and **consistent-prefix**. The full set of four is:
- **Monotonic writes** — your *own* writes are applied in the order you issued them (write A then B → B never lands before A).
  A single leader preserves this naturally, since it serializes one client's writes in arrival order.
- **Writes-follow-reads** — if you read X and then write Y (a reply that depends on X), anyone who sees Y is guaranteed to
  also see X. Keeps cause-before-effect intact (e.g. no one sees your comment on a post they can't yet see).

### 2. CRDT canonical examples (by type)
- **Counters:** **G-Counter** (grow-only — increments only) and **PN-Counter** (Positive/Negative — supports both
  increment and decrement).
- **Sets:** **OR-Set** (Observed-Remove Set — supports add *and* remove without an add "losing" to a concurrent remove).
- This is the machinery behind **Google-Docs-style collaboration** and **shopping-cart merges** — concurrent edits on
  different replicas merge automatically with **no lost data** and no manual conflict resolution.

### 3. The durability / latency / availability triangle (replication modes)
- **Sync** — buys **durability**, pays in **latency + availability** (a write blocks on a slow/down replica; if it stalls,
  you stall).
- **Async** — buys **latency + availability** (leader never waits), pays in **durability** (a lost-writes gap on failover —
  the un-shipped tail is gone).
- **Semi-sync** — the pragmatic middle: wait for **one** ack (ideally a region-local replica), rest async. Most of the
  durability, little of the latency. (See F-2 above.)

### 4. Stale-read caveat (async replicas)
- Reads served off an **async replica can be stale** (it's behind by the replication lag). Never route **read-after-write
  critical** data — a just-placed order, an account balance, a just-updated setting — to a lagging replica; the user reads
  their own action as "missing" and panics.
- **Fix:** route those specific reads to the **leader**, or use **read-your-writes routing** (pin a user to a replica
  known to be caught up past their last write). Non-critical reads can happily stay on the async replicas.

---

## ⭐ Completeness additions (pass 2)

### 5. Semi-sync in real systems — what it's actually called
Interviewers like it when you can name the real knob, not just the concept:
- **MySQL — "semisynchronous replication."** The source waits for at least one replica to say "I've received
  and logged this change" before telling the client "committed" (how many is a setting, default 1). ⚠️ The
  catch: if no replica answers within a timeout (default 10 seconds), MySQL **quietly falls back to async**
  so writes don't hang forever — durability silently weakens until a replica catches up.
- **PostgreSQL — "synchronous replication."** You list standbys in `synchronous_standby_names`, e.g.
  `FIRST 1 (replica_a, replica_b)` ("wait for the highest-priority standby in this list that is currently connected" — replica_a if it's up, otherwise replica_b) or `ANY 1 (...)` ("wait for
  any one"). Waiting for 1 of N is exactly the semi-sync idea. Unlike MySQL, if no synchronous standby is
  available, Postgres commits **wait** rather than downgrade — it chooses durability over availability.
- **MongoDB** — the same idea is expressed per write as **write concern**: `w:1` (primary only, async-ish),
  `w:2` (primary + one secondary ≈ semi-sync), `w:"majority"` (quorum).

### 6. Failover — the full list of what can go wrong
Section D covered lost writes and split-brain. A real failover has more traps (DDIA Ch. 5 lists these):
1. **Lost writes (async).** Covered above. Worse, the discarded writes can still have **side effects
   elsewhere**. Real example DDIA cites (GitHub): a lagging MySQL follower was promoted and re-issued
   auto-increment IDs the old leader had already handed out; those IDs were also used as keys in Redis, so
   some private data was shown to the **wrong users**. Lesson: the DB "forgetting" writes isn't contained to the DB.
2. **Split-brain.** Covered above — fix with majority vote + fencing tokens.
3. **Choosing the timeout is a no-win dial.** Declare the leader dead too **quickly** and a brief load spike or
   network hiccup triggers a needless failover (which adds load and can cascade). Too **slowly** and you sit
   with no writes for that whole period.
4. **Everyone must learn who the new leader is.** Clients, proxies, and the other followers must all be
   repointed (service discovery / a virtual IP / a coordination service like ZooKeeper or etcd). Stragglers
   keep writing to the old address.
5. **The old leader must rejoin as a follower** — and throw away any writes the new leader never got
   (that's where lost writes physically disappear).
- **Interview one-liner:** "Failover risks: lost async writes (with side effects outside the DB), split-brain,
  a timeout that's either trigger-happy or slow, and clients still pointing at the old leader — so I use a
  majority-based election, fencing tokens, and semi-sync for at least one durable copy."

### 7. Quorum dials at the extremes, and when `W + R ≤ N`
Section E gave the rule. Now the three standard settings for **N = 3**, so you can answer "tune it for X":
- **W = 3, R = 1 (W = N)** — every write must reach all 3 copies, so any single copy is fresh → **fastest
  reads**. Cost: one dead replica blocks all writes.
- **W = 1, R = 3 (R = N)** — writes return after 1 ack → **fastest writes**; reads must check all 3 to be sure
  of the latest. Cost: one dead replica blocks all strong reads.
- **W = 2, R = 2** — balanced; tolerates one dead replica for both reads and writes. The usual default.
- **W = 1, R = 1 (`W + R ≤ N`)** — 1 + 1 = 2, not > 3. The write might land only on node A while the read asks
  only node C → you **read stale data**. That's a legitimate choice (it's eventual consistency, fast and
  highly available) — just know you've given up the overlap guarantee.
- ⚠️ **Caveats even when `W + R > N`:** (1) a **sloppy quorum** — when the "home" nodes are down, the write is
  accepted by *other* nodes as hints (hinted handoff) — means the read set may not overlap the nodes holding
  the write until the hints are delivered; (2) concurrent writes and partially-failed writes can still produce
  edge cases. DDIA's summary: quorums make stale reads **unlikely**, not impossible — they don't by themselves
  give linearizability.

### 8. Replication methods — which real systems use which
Builds on Extras above (statement / physical WAL / logical row-based):
- **PostgreSQL streaming replication** ships the **physical WAL** → replicas must run the same major version;
  Postgres **logical replication** ships row changes and works across versions (and powers CDC tools).
- **MySQL** replicates via its **binary log (binlog)**, which can be `STATEMENT`, `ROW`, or `MIXED` format;
  **ROW** has been the default since MySQL 5.7, precisely because statement-based replays of things like
  `NOW()` or `RAND()` can diverge.
- **MongoDB** replica sets copy an **oplog** — a log of logical, idempotent operations — from primary to
  secondaries.
- **CDC** (Change Data Capture, Day 19) reads these logs (e.g. Debezium reading the Postgres WAL or MySQL binlog)
  to stream DB changes to Kafka, caches, search indexes.

### 9. Real-system map — additions
- **MongoDB replica set** → single-leader (one primary, secondaries), automatic election by majority vote.
- **Spanner / CockroachDB** → per-range consensus groups (Paxos / Raft): every write is a majority-quorum
  write, so they're "single-leader per shard + quorum" rather than async primary-replica. (See the F-2
  correction for how replica placement decides write latency.)
