# Day 11 — Message Queues & Event Streaming
**Phase 1: Foundations** | Score: 9.5/10

> **One-sentence idea:** A message queue lets one service **hand off work to another without waiting**
> — the sender drops a message in the middle and moves on; the receiver picks it up when ready. Like a
> **restaurant ticket rail**: the waiter clips an order and goes back to serving; the kitchen pulls
> tickets at its own pace. Neither blocks the other, and if the kitchen falls behind, tickets queue up
> instead of being lost. Everything today flows from this **decoupling in time** — and its consequences:
> what if a message is delivered twice? out of order? what if the consumer crashes mid-ticket?

## 1. Why async messaging exists (3 problems it solves)
Without a queue, A calls B directly (sync HTTP) and **waits**. At scale, 3 failures:
1. **Tight coupling** — if B is down/slow, A is down/slow too. A queue decouples: A drops a message and
   succeeds even if B is dead; B processes later.
2. **Traffic spikes** — 10,000 orders/sec but payment handles 500/sec → direct calls drop 9,500. A queue
   does **load leveling (buffering)**: absorbs the spike, consumer drains steadily. A shock absorber.
3. **Slow work blocks the user** — email / thumbnail / report takes seconds. Drop a message, return
   "200 OK" instantly, do the slow work **asynchronously** in the background.
- The trade: give up the *instant in-line answer* (sync) for *resilience + smoothing + responsiveness*
  (async). Cost = complexity (§6–§10).

## 2. Core vocabulary
- **Producer** sends · **Consumer** receives/processes · **Broker** = the middleman server holding
  messages (RabbitMQ/Kafka/SQS) · **Message** = one unit (event or command) · **Queue** = ordered
  holding area · **Topic** = named category/channel consumers subscribe to.

## 3. Two messaging MODELS (who receives each message)
- **Point-to-point (work queue / competing consumers)** — each message → **exactly ONE** consumer. Run 5
  workers on one queue → they compete, each message handled once → **scale out** by adding workers.
  (One ticket rail, five cooks.) **Splits the work.**
- **Publish/subscribe (pub/sub, fan-out)** — a message on a **topic** → **EVERY** subscriber gets a copy.
  "OrderPlaced" fans out to email + analytics + inventory, each its own copy. (Radio broadcast.) **Copies the work.**

## 4. ⭐ The big structural split: traditional Queue vs Log/Stream (= "RabbitMQ vs Kafka")
- **Traditional queue (RabbitMQ, SQS)** — a message is a **letter**: once read + acked, the broker
  **deletes it**. Temporary holding area, no going back. Great for one-time task/job commands.
- **Log / event stream (Kafka)** — messages **appended to an immutable, durable log** (like Day 10's
  LSM append). Reading does **NOT** delete — kept for a **retention period** (e.g. 7 days). Each consumer
  tracks its own **offset** (bookmark: "read up to #1,050"). Two superpowers:
  - **Replay** — a new/recovering consumer rewinds offset to 0 and re-reads all history.
  - **Multiple independent consumers** — analytics at offset 900, billing at 1,050, same log, each its own pace.
- Analogy: queue = **to-do list you cross off and discard**; log = **diary you append to and can re-read**.
- ⭐ The decision: need durable history / replay / high-throughput streaming / many consumers of the same
  data → **log (Kafka)**. Need flexible routing + simple delete-after-done work queue → **RabbitMQ**.

## 5. Push vs Pull (how the message reaches the consumer)
- **Push (RabbitMQ)** — broker pushes as messages arrive. Low latency, but can overwhelm a slow consumer
  (needs a *prefetch* limit).
- **Pull (Kafka, SQS)** — consumer polls when ready → controls its own pace → natural **backpressure**
  (pulls slower when busy). Slightly higher latency, but can't be flooded → scales to huge throughput.

## 6. ⭐⭐ Delivery semantics (the most-tested subtopic)
Because networks drop packets and consumers crash mid-process, "deliver this" has 3 guarantees:
- **At-most-once** — deliver, don't retry. May **drop** messages. Fast/simple. OK for a throwaway metric.
- **At-least-once** — retry until acked. **Never lost, but may arrive >once** (consumer processed it, crashed
  before acking → broker redelivers). **The practical default.**
- **Exactly-once** — once, no loss, no dup. The holy grail — **genuinely hard**: "process it" and "ack it"
  can't be made perfectly atomic across a network. (This is the **Two Generals problem**: over an unreliable link, the
  sender can never be *certain* its last message — the ack — arrived, so one side must either risk a duplicate or risk a loss.)
- ⭐ **THE KEY INSIGHT (Q3 — I nailed this):** make at-least-once *behave like* exactly-once with an
  **idempotent consumer** — processing the same message twice has the same effect as once.
  - How: unique message ID; consumer records handled IDs (DB/Redis) and **skips duplicates**. Or design a
    naturally idempotent op (`SET balance=100` is idempotent; `balance = balance+100` is not).
  - Interview gold: "True exactly-once delivery is nearly impossible; the real answer is **at-least-once +
    idempotent consumer** = exactly-once *effect*." (Kafka's "exactly-once semantics" holds *within* Kafka
    via idempotent producers + transactions — but a side effect that leaves Kafka, like charging a card,
    STILL needs consumer idempotency.)

## 7. Ordering guarantees — partitions ⭐ (Q4 — my naming miss)
Order is **not** guaranteed globally, because throughput needs **parallelism**, and parallelism fights order.
- **Kafka partitions:** a **topic** splits into **partitions** = independent ordered logs. Order is
  guaranteed **only within one partition**, never across.
- ⭐ **THE MECHANISM (name it precisely):** a message's **partition key** decides its partition via
  `hash(key) % numPartitions` (Day 9 sharding idea). **Use `order_id` as the partition key** → every event
  for one order lands in the **same partition** → in-order. Different orders scatter → parallel. So:
  **"same partition key → same partition → ordered."** ⚠️ I said "some technique to allocate it" — name it: **partition key.**
  - Trade: more partitions = more parallelism/throughput, but order only holds within each.
- **Consumer group** = a set of consumers sharing a topic's work. Kafka assigns **each partition to exactly
  one consumer in the group** → parallel + per-partition order. Max useful consumers = number of partitions
  (extras idle). Adding/removing consumers → a **rebalance**.
- SQS **FIFO** preserves order within a "message group ID"; SQS **Standard** = best-effort order only.

## 8. Acknowledgements / offsets / visibility timeout (not losing work)
- **Ack/nack (RabbitMQ, SQS)** — consumer acks after success → broker deletes. Crash before ack →
  **redelivered** (source of at-least-once dups). Nack = "failed, requeue."
- **Offset commit (Kafka)** — consumer commits offset ("processed up to #1,050"); on restart resumes there.
- **Visibility timeout (SQS)** — a pulled message goes **invisible** to others for N sec while processed.
  Ack in time → deleted. Crash/timeout → reappears for another worker. (Stops double-grab, still recovers.)
- ⭐ Subtle bug: ack/commit **AFTER** processing. Ack-before = at-most-once (crash loses it);
  process-before-ack = at-least-once (crash redelivers).

## 9. Reliability machinery
- **Durability** — brokers **write to disk + replicate** across nodes (Kafka replicates each partition; SQS
  managed-redundant) so a crash doesn't lose data. In-memory-only = fast but loses on crash.
- **Dead Letter Queue (DLQ)** ⭐ (Q5 — I nailed this) — a repeatedly-failing **poison message** (malformed /
  triggers a bug) would be retried forever, blocking the queue. After N attempts the broker moves it to a
  separate **DLQ** for human inspection; the main queue keeps flowing.
- **Retries + backoff** — retry failures with **exponential backoff + jitter** (don't hammer a struggling
  downstream — Day 21 resilience).
- **Backpressure / load leveling** — queue depth *is* the buffer; if it grows unbounded the consumer can't
  keep up → alarm on depth, autoscale consumers, or shed load.

## 10. ⭐ The dual-write problem → Outbox pattern (Day 19 preview)
- Trap: a service must **(a) save to its DB** and **(b) publish a message** — two systems, no shared txn. Save
  then crash before publishing → DB has the order but no message ever sent → downstream permanently out of sync.
- **Outbox pattern:** write the message into an `outbox` table **in the same DB transaction** as the order
  (atomic — both or neither). A separate process reads the outbox and publishes (often via CDC, Day 8). The
  message is never lost relative to the DB write. (Just know the *name* + that the problem is real.)

## 11. The big three
| | **Kafka** | **RabbitMQ** | **Amazon SQS** |
|---|---|---|---|
| Type | Distributed **log/stream** | Traditional broker (AMQP) | Managed queue (cloud) |
| Model | Pull, partitioned log, offsets | Push, **exchange→binding→queue** | Pull, fully managed |
| Deletes on read? | No — retained, **replayable** | Yes — after ack | Yes — after ack |
| Ordering | Per-partition | Per-queue | Standard: best-effort · **FIFO: ordered** |
| Throughput | **Very high** (M/sec) | Moderate-high | High (managed) |
| Superpower | Replay, retention, fan-out, streaming | **Flexible routing** | **Zero ops** (AWS runs it) |
| Best for | Event streaming, log pipelines, analytics | Complex routing, task queues, RPC | "Just give me a queue" on AWS |
- **RabbitMQ routing:** producer → **exchange** → routes to queues via **bindings**: *direct* (exact key),
  *topic* (wildcards), *fanout* (broadcast), *headers* (match on message header attributes instead of the routing key).
  Routing flexibility = its signature strength.
- **SQS:** **Standard** = at-least-once + best-effort order + huge throughput; **FIFO** = exactly-once
  processing + strict order, lower throughput.

## 12. When to use a queue — and when NOT
- **Use:** work can be async, absorb spikes, decouple services, fan out events, background processing.
- **Don't:** when you need an **immediate synchronous answer** ("is this password correct?") — a queue adds
  latency + complexity. And not "just because" — it's real ops weight (a broker to run + duplicate/order reasoning).

---

## 🎯 The task & my answers (e-commerce order pipeline, spiky traffic)
1. **Why a queue:** absorb the spike (**load leveling**) + **decouple** services + **slow work doesn't block
   the user** (async). ✅
2. **Model:** **pub/sub** — one OrderPlaced event → payment + email + inventory + analytics each get a copy.
   Use a **log/stream** (immutable, consumers advance their own offset, replayable). ✅
3. **Delivery:** **at-least-once + idempotent consumer** — retries on failure; consumer dedups by a unique ID
   (order_id / transaction_id) so a redelivery never double-charges. ✅
4. **Ordering:** Kafka — partition by **`order_id` as the partition key** → all events for one order hit the
   same partition → in-order; different orders parallel. ⚠️ I said "some technique" — name it: **partition key.**
5. **Failure:** the **poison message** is moved to a **DLQ (dead letter queue)** after N failed attempts, so
   it stops jamming the main queue. ✅

---

## ⭐ Follow-up (RESOLVED) ✅
**Q:** Payment service writes `status='charged'` to its DB, then crashes **before** publishing
`PaymentSucceeded` to Kafka → DB says charged but no downstream service knows. What is this problem called,
and what named pattern fixes it?
**A:** The **dual-write problem** (two systems, no shared transaction → crash between them = permanent
inconsistency). Fixed by the **Outbox pattern**: write the message into an `outbox` table **in the SAME DB
transaction** as the status update (atomic — both or neither), then a **background relay** reads the outbox
and publishes to Kafka (often via CDC, Day 8). The message can never be lost relative to the DB write.
✅ Answered correctly *and named the mechanism* (single transaction + background relay).

---

## Personal Notes (recurring gaps)
- [ ] ⭐ **#1 CRAFT GAP NOW — name the precise mechanism, don't describe it.** Q4: I said "some technique to
  allocate messages to partitions" instead of "**use order_id as the partition key**." I *know* it; in an
  interview, describing-instead-of-naming loses the point. Same half-point lost on Day 9 (access skew) and Day 10
  (read amplification). Drill: when you feel yourself saying "some technique/thing," STOP and name it.
- [x] **Delivery semantics 3/3** ✅ (at-least-once + idempotent consumer + dedup ID — the hardest part).
- [x] **Queue vs log distinction + pub/sub correct** ✅.
- [x] **Poison message + DLQ** ✅.

---

## ⭐ Interview-completeness additions (audit pass)

These are the follow-up questions an interviewer reaches for *after* you've nailed the core. Close them and you're airtight.

### 1. "Why Kafka over Kinesis?" (know the AWS twin)
**Amazon Kinesis** is AWS's managed Kafka-equivalent — same core idea: a **partitioned, replayable log**.
- Partitions are called **SHARDS**; each has a **retention window** (24h default, up to **365 days**).
- Ordering is **per-shard**, chosen by a **partition key** — same mental model as Kafka.
- **Per-shard caps: 1 MB/s in, 2 MB/s out.** Scaling = adding shards.

**When to pick which:**
- **Kinesis** → you're already on AWS and want **zero-ops** (no brokers to run).
- **Kafka** → **higher throughput**, **richer ecosystem** (Kafka Connect, Kafka Streams), and **no per-shard rate caps**.

### 2. "How do you NOT lose messages when a broker crashes?" — acks + ISR + replication
Three knobs that together give you durability guarantees:
- **Replication factor** — how many brokers hold a copy of each partition (e.g. **3**).
- **ISR (in-sync replicas)** — the replicas currently caught up to the leader.
- **Producer `acks`:**
  - `acks=0` — fire-and-forget. Fastest, **can lose data**.
  - `acks=1` — leader confirms only. **Lost if the leader dies before replicating.**
  - `acks=all` — wait for **all in-sync replicas**. **Safest.**

**The safe combo:** `replication factor = 3` + `min.insync.replicas = 2` + `acks=all`
→ a write **survives one broker failure with zero data loss** (2 replicas still have it).

### 3. Consumer lag — the health metric to name
> **lag = latest offset produced − offset the consumer has committed**

= how far behind real-time the consumer is. **Rising lag = consumers can't keep up.**
Fix: **autoscale consumers** (only useful **up to the partition count** — one consumer per partition max) or fix a **slow downstream** (DB, API) that's the real bottleneck.

### 4. Retention vs log compaction — two ways to bound the log
- **Time/size retention** (`retention.ms` / `retention.bytes`) — drop **old segments wholesale**. Good for **event streams** where old events stop mattering.
- **Log compaction** — keep only the **LATEST value per key**, discarding superseded updates. Turns the log into a **replayable snapshot of current state** (e.g. latest address per user). This is what makes a topic usable as a **source of truth / changelog**.

### 5. Fan-out on write vs on read (preview — feeds/notifications)
- **Fan-out-on-write** — push a copy to **each consumer/feed at publish time**. Fast reads; **expensive for celebrities** (millions of copies per post).
- **Fan-out-on-read** — compute the feed **at query time**. Cheap writes; heavier reads.
Pub/sub is the **on-write mechanism**. This trade-off reappears in **feed / notification** design.

### 6. Event sourcing (Phase 2 preview)
Instead of storing **current state**, store the **immutable append-only log of events** and **derive state by replaying them** — the log/stream model taken to its logical end. Pairs with **CQRS** (separate write model and read model).

### 7. Claim-check pattern (brief)
Don't put **large blobs** in messages. Store the blob in **S3 / object storage** and put a **reference/pointer** in the message. Keeps the queue fast and within message-size limits.

---

## ⭐ Completeness additions (pass 2)

### 8. How a producer picks a partition — and why you can't casually add partitions
- **With a key:** Kafka's default partitioner computes `murmur2(key) % number_of_partitions` → the same key always lands on
  the same partition → per-key ordering (§7). This is Day 9's `hash % N`, inside Kafka.
- **Without a key (`key=null`):** there's no ordering to protect, so the producer just **spreads** records across
  partitions (modern clients fill a batch for one partition, then move to the next — "sticky" partitioning — for efficiency).
- ⚠️ **Partition count can only go UP.** Kafka can add partitions to a topic but **cannot reduce** them (you'd recreate the topic).
- ⚠️ **Adding partitions breaks key → partition mapping.** Go from 6 → 8 partitions and `hash(order-1001) % 8` usually
  differs from `% 6`. New events for that order land on a **different** partition than its old events, so a consumer can
  see them **out of order** around the change. Existing data is **not** moved. This is Day 9's "`% N` trap" again —
  Kafka doesn't use consistent hashing here.
- **So:** pick the partition count **up front with headroom** (it caps your consumer parallelism — §7: max useful consumers =
  partitions). If you must grow a keyed, order-sensitive topic, the safe route is a **new topic** with more partitions and a
  planned migration (drain the old one, switch producers).

### 9. Consumer groups & rebalancing — what actually happens
**The idea:** a consumer group is a team splitting the partitions between them. A **rebalance** is the team re-dividing the
work whenever the team changes.
- **Triggers:** a consumer **joins** (you scaled up), **leaves** cleanly (shutdown), or is **declared dead** — it stopped
  sending heartbeats, or took too long between `poll()` calls (its processing loop got stuck; `max.poll.interval.ms`).
  Also when partitions are added to the topic.
- **Classic ("eager") rebalance = stop-the-world:** every consumer gives up *all* its partitions, then everything is
  re-assigned. Nobody in the group processes anything during the pause.
- **Cooperative (incremental) rebalance** — modern assignors (e.g. `CooperativeStickyAssignor`) move **only the partitions that
  need to move**; everyone else keeps working.
- **Why rebalances create duplicates:** a consumer was halfway through partition 3 and hadn't committed offsets for its last
  few messages. Partition 3 moves to another consumer, which resumes from the **last committed** offset → those few
  messages are processed **again**. One more reason the consumer must be **idempotent** (§6).
- **Avoiding needless rebalances:** keep per-message processing short (or hand slow work to a thread pool and keep
  polling); for rolling restarts, **static membership** (`group.instance.id`) lets a restarting consumer reclaim its
  partitions without triggering a full rebalance.

### 10. Offset commits — auto vs manual (and where offsets live)
- Committed offsets are stored by Kafka itself in an internal topic, **`__consumer_offsets`**, per (group, partition).
- **Auto-commit** (`enable.auto.commit=true` — the default; every `auto.commit.interval.ms`, default 5 s): the client
  periodically commits "whatever I've polled so far". Easy, but the commit is **not tied to your processing**:
  - Crash 4 s after the last auto-commit → the last ~4 s of messages are **redelivered** (duplicates).
  - If you process asynchronously (hand messages to other threads), the offset can be committed **before** processing
    finishes → crash → those messages are **lost** (silent at-most-once).
- **Manual commit** (`enable.auto.commit=false`): **you** call `commit()` **after** processing succeeds → clean
  **at-least-once** (the ack-after-processing rule in original §8). Use it for anything that matters (payments, orders).
- **`auto.offset.reset`** — what to do when the group has **no** committed offset yet (brand-new group): `latest` (default —
  only new messages from now on) or `earliest` (replay the whole retained log).

### 11. 🛠️ Hands-on: a minimal Kafka producer + consumer (Python, `confluent-kafka`) — Rule F
**Step 1 — create the topic** (6 partitions, 3 copies of each):
```
kafka-topics.sh --create --topic orders --partitions 6 --replication-factor 3 --bootstrap-server localhost:9092
```
**Step 2 — producer:**
```python
from confluent_kafka import Producer

producer = Producer({
    "bootstrap.servers": "localhost:9092",
    "acks": "all",                    # wait for all in-sync replicas (addition 2)
    "enable.idempotence": True,       # broker drops duplicates caused by producer retries (pass-2 §12)
})

def on_delivery(err, msg):            # called once per message when the broker answers
    if err is not None:
        print(f"delivery FAILED: {err}")          # alert / retry / store for later
    else:
        print(f"stored in {msg.topic()}[{msg.partition()}] at offset {msg.offset()}")

producer.produce(
    "orders",
    key="order-1001",                 # partition key → all events of this order share one partition (ordered)
    value='{"order_id":"order-1001","event":"OrderPlaced","total_cents":4999}',
    on_delivery=on_delivery,
)
producer.flush()                      # block until every buffered message is acknowledged
```
- `produce()` is **asynchronous** — it only puts the message in a local buffer; the client batches and sends it.
  That's why there's a delivery callback, and why you `flush()` before exiting (or messages in the buffer are lost).
- `key` decides the partition (pass-2 §8); `value` is just bytes — JSON here.

**Step 3 — consumer (manual commit, idempotent):**
```python
import json
from confluent_kafka import Consumer

consumer = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "payment-service",    # consumers with the same group.id split the partitions
    "enable.auto.commit": False,      # WE commit, only after processing succeeds
    "auto.offset.reset": "earliest",  # new group → start from the oldest retained message
})
consumer.subscribe(["orders"])

try:
    while True:
        msg = consumer.poll(1.0)              # wait up to 1 s for a message
        if msg is None:
            continue                          # nothing yet
        if msg.error():
            print(msg.error())
            continue
        event = json.loads(msg.value())
        if not already_processed(event["order_id"]):     # idempotent consumer: dedup by business ID
            charge_card(event)                           # the side effect
            mark_processed(event["order_id"])            # ideally in the SAME DB transaction as the side effect
        consumer.commit(message=msg, asynchronous=False) # commit AFTER processing → at-least-once
finally:
    consumer.close()                          # leave the group cleanly → fast rebalance
```
- Line by line: `group.id` puts this process in the team; `subscribe` asks for the topic (the group decides which
  partitions *this* process gets); `poll` fetches (pull model, §5); the `already_processed` check makes redeliveries
  harmless; `commit` after the work = at-least-once; `close()` triggers an orderly rebalance.
- `already_processed` / `mark_processed` are your own functions — e.g. a `processed_events` table with a **UNIQUE** constraint
  on the ID (same durable-dedup idea as Day 12's resolved follow-up).

**Step 4 — run two copies of the consumer** and watch the 6 partitions split 3/3 (a rebalance). Kill one → its 3
partitions move to the survivor.
**Step 5 — check lag:**
```
kafka-consumer-groups.sh --describe --group payment-service --bootstrap-server localhost:9092
```
Shows, per partition, `CURRENT-OFFSET` (committed), `LOG-END-OFFSET` (latest), and `LAG` (the difference — addition 3).

### 12. Idempotent producer (and Kafka transactions) — duplicates on the WRITE side
- **The problem:** the producer sends a message, the broker stores it, but the **ack is lost** → the producer retries → the
  log now holds the message **twice**. (§6 handled duplicates on the *consumer* side; this is the *producer* side.)
- **The fix:** with `enable.idempotence=true` the broker gives each producer a **producer ID**, and the producer numbers its
  messages per partition (**sequence numbers**). If the broker receives a sequence number it has already written, it
  **drops the duplicate** (and still acks). Retries become safe. In the **Java client**, `acks=all` is the default since Kafka 3.0, and idempotence is on by default since **3.0.1 / 3.1.1 / 3.2.0** (a bug kept it off in 3.0.0 and 3.1.0) — as long as no conflicting setting (e.g. `acks=1`) is present. ⚠️ **librdkafka** — the engine under `confluent-kafka` (the Python code above), Go and .NET clients — still defaults `enable.idempotence=false` (its `acks` default is already all), which is exactly why the snippet sets it explicitly. Rule: **always set it yourself; never rely on a client default.**
- **Limits:** it covers retries of one producer session writing to Kafka. It does **not** stop your *application* from calling
  `produce()` twice for the same business event, and it does nothing for side effects outside Kafka.
- **Kafka transactions** go one step further: write to **several partitions** (and commit consumer offsets) **atomically** —
  all or nothing — for consume → transform → produce pipelines (Kafka Streams' "exactly-once"). Consumers must use
  `isolation.level=read_committed` to skip aborted writes. Still only *inside* Kafka (§6's caveat stands).

### 13. Retries done right — exponential backoff + jitter, concretely
**Step 1 — only retry what can succeed later.**
- **Retryable (transient):** timeouts, connection refused, `503`, `429`, a DB deadlock.
- **Not retryable (permanent):** malformed JSON, validation failure, `400`/`404`, "card declined". Retrying a poison
  message just burns time → send it **straight to the DLQ**.

**Step 2 — wait longer each time (exponential backoff).** Retrying instantly hammers a downstream that's already
struggling. Double the wait each attempt: 100 ms, 200 ms, 400 ms, 800 ms … up to a **cap** (e.g. 10 s), and a **max
number of attempts** (e.g. 5) before giving up → DLQ.

**Step 3 — add jitter (randomness).** If 10,000 clients all failed at the same instant, plain backoff makes them all retry
at the same instants too (+100 ms, +200 ms …) — synchronized waves that knock the service over again (a "thundering herd").
**Full jitter** (from the well-known AWS Architecture Blog analysis) picks a **random** wait between 0 and the backoff:
```python
import random, time

def call_with_retries(fn, max_attempts=5, base=0.1, cap=10.0):
    for attempt in range(max_attempts):
        try:
            return fn()
        except TransientError:
            if attempt == max_attempts - 1:
                raise                                   # out of attempts → caller sends to DLQ
            backoff = min(cap, base * 2 ** attempt)     # 0.1, 0.2, 0.4, 0.8 … seconds, capped
            time.sleep(random.uniform(0, backoff))      # full jitter: spread clients out randomly
```
- `TransientError` stands for "the retryable kinds" — permanent errors are not caught, so they fail fast.
- Deeper treatment (circuit breakers, retry budgets) → **Day 21**.

### 14. Head-of-line blocking — why a stuck message hurts a whole partition
- **The idea:** in an ordered lane, the item at the **front** blocks everyone behind it — like one slow car on a single-lane road.
- **In Kafka:** a consumer processes a partition **in order** and commits offsets in order. If message #500 keeps failing
  and you retry it *in place* (sleep, retry, sleep …), messages #501, #502 … in that partition all **wait** — even though
  they're for completely different orders. Lag on that partition climbs while others are fine.
- **Fixes:**
  - **Retry topics (non-blocking retries):** on failure, publish the message to `orders-retry-1` (consumed after a delay), then
    `orders-retry-2` (longer delay), then the **DLQ**; commit the original offset and **move on**. The main lane keeps flowing.
    Trade: that one message is now processed **out of order** relative to its key — fine for independent events, not
    for strictly ordered ones (for those you must block, or park all later events of the *same key* too).
  - **Bounded retries in place** (e.g. 3 quick tries with backoff) before diverting.
- Same idea in classic queues: SQS/RabbitMQ use a **max-receive-count → DLQ** policy so a poison message can't block forever.

### 15. Hot partitions — diagnosis and fixes
- **Symptom:** one partition's lag climbs while the others are near zero; one consumer is pegged at 100% CPU, the rest idle.
- **Cause:** **key skew** — one key (a giant merchant, a celebrity user, `country=US`) produces a huge share of events, and
  every event with that key goes to one partition (= one consumer). Adding consumers **doesn't help** — one partition can
  only be read by one consumer in the group. (Day 9's access skew, in a queue.)
- **Fixes:**
  - **Better key:** use a higher-cardinality key (`order_id` instead of `merchant_id`) if per-merchant ordering isn't truly needed.
  - **Salting / compound key:** `merchant_42#0 … merchant_42#7` spreads that merchant over 8 partitions. You keep
    ordering **within each sub-key** only — acceptable when ordering is per order, not per merchant.
  - **Dedicated topic** for the whale, with its own consumers.
  - **Speed up the consumer** for that partition (batch DB writes, parallelize per key inside the consumer while
    preserving per-key order).
  - **Backpressure/throttle** the noisy producer (quotas) if it's abusive rather than legitimate.

### 16. Backpressure — the concrete toolkit
Backpressure = **the slow side signals "slow down" to the fast side** instead of silently drowning.
- **Pull-based consumption** (Kafka, SQS) is natural backpressure: a busy consumer simply polls less (§5).
- **Prefetch limits** for push brokers: RabbitMQ's prefetch count = "send me at most N unacked messages" — the broker
  stops pushing when the consumer is full.
- **Bounded buffers:** an in-memory queue inside your service should have a **max size**; when full, block the producer
  or reject (e.g. HTTP **`429`/`503`** to callers) — never grow until the process runs out of memory.
- **Autoscale consumers on lag/queue depth** (e.g. SQS `ApproximateNumberOfMessagesVisible`, Kafka consumer lag) — but
  only up to the partition count for Kafka.
- **Load shedding:** drop or defer low-priority work (analytics) to protect critical work (payments).
- Kafka's durable log is itself a huge buffer — days of retention mean a slow consumer falls behind *without data loss*,
  then catches up.

### 17. CDC as the outbox relay (finishing §10)
- **CDC (Change Data Capture)** reads the database's own change log — Postgres **WAL**, MySQL **binlog** — and turns every
  committed insert/update/delete into an event. Tool to name: **Debezium** (streams DB changes into Kafka).
- **Outbox + CDC:** your transaction writes the order row **and** an `outbox` row; Debezium tails the log, sees the committed
  outbox row, and publishes it to Kafka. No polling loop, no dual write, and the event appears **only if** the transaction
  committed. Delivery is still at-least-once → consumers stay idempotent. Deep dive → **Day 19**.

### 18. SQS specifics worth knowing
- **Visibility timeout** (original §8) must be **longer than your processing time**; otherwise the message reappears while you're still
  working on it → a second worker processes it too.
- **FIFO queues:** ordering per **message group ID**; deduplication uses a **deduplication ID** (or a hash of the body),
  and a duplicate sent within the **5-minute deduplication interval** is accepted but not delivered again.
- **Redrive policy:** `maxReceiveCount` → after that many failed receives, SQS moves the message to the DLQ.
