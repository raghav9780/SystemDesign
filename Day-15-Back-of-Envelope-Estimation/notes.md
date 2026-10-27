# Day 15 — Back-of-the-Envelope Estimation
**Phase 1: Foundations (final day)** | Score: 8.5/10 | Hands-on topic

> **One-sentence idea:** Estimation = **rough capacity math — QPS, storage, bandwidth, memory, servers — done in
> your head with round numbers** to prove a design is grounded in reality and to expose the **bottleneck** before
> building anything.
>
> Analogy: a **builder pacing out a plot of land** before drawing the house. They don't need the area to the
> centimetre — they need to know "is this a cottage or a tower block?" because that decides the foundation.
> Estimation tells you whether you're building a cottage (one DB, one box) or a tower block (shards, caches, CDN).

- Goal is the **right order of magnitude**, never precision. Off by 2× = fine. Off by 1000× (GB vs TB) = fail.
- Modern rule: estimate **only when the number changes a decision** ("fits in one Redis, or must I shard?"), not as ritual.
- The payoff is always the **"so what"** → which resource is the bottleneck, and what architecture that forces.

---

## 0. Why estimate at all — and why ONLY when it changes a decision
Every design choice you make in an interview is secretly a bet about size. "Put it in Postgres" is a bet that the
data fits on one machine and the write rate fits one leader. "Add a cache" is a bet that reads dominate. The
estimate is how you **check the bet out loud** instead of guessing.

**Mini-example.** You're designing a URL shortener. You *could* spend 3 minutes computing exact bandwidth. But
ask: would any answer change your design? If bandwidth comes out at 400 Mbps, that's a single NIC's worth — no
decision changes. If **storage** comes out at ~270 TB, that *does* change a decision: it can't live on one
machine → you must shard. So you spend your time on storage and wave through bandwidth in one line.

**Why the modern rule exists:** interviews are ~45 minutes. Candidates who compute every number "because the
template says so" burn 10 minutes and still never say what the numbers *mean*. Interviewers in 2025-26 increasingly
say "skip the math unless it matters." The skill is knowing **which** number is the decision-maker.

⭐ **Interview one-liner:** "I'll estimate the numbers that decide the architecture — mainly whether storage and
peak QPS fit on one node — and skip the ones that don't change anything."

➡️ **Where this sits in an answer (Day 16 — answer framework):** requirements first (functional + non-functional),
*then* a quick estimate **only where it changes a decision**, then high-level design. Estimation is a step in the
framework, not the whole answer.

---

## 1. The 4-step framework (the spine — say it out loud)
1. **Clarify assumptions** — state DAU, actions/user, payload sizes, retention out loud, so the interviewer
   corrects an *input*, not your *method*. ("I'll assume 100M DAU — tell me if you had a different scale in mind.")
2. **Approximate aggressively** — round everything to one significant figure and to powers of 10. 86,400 → 10⁵.
   You lose ~14% accuracy and gain the ability to do it in your head with no mistakes.
3. **Break down, then aggregate** — compute per-user / per-day first (small, checkable numbers), then scale up.
4. **Sanity-check** — does the magnitude feel right? "2 million GB for a chat app's daily messages?" — that's
   2 PB/day, more than it should be → you just caught a 1000× slip. (This is exactly the slip I made in Task 2.)

**Why stating assumptions first matters:** if you silently assume 1B DAU and the interviewer meant 10M, every
number after is 100× off and it looks like your *math* is wrong. Say the input out loud and they'll correct the
input — your method stays intact.

---

## 2. Numbers to memorize

### 2.1 Powers of two (data sizes) vs powers of ten (counts)
A kilobyte is technically 2¹⁰ = 1,024 bytes, but 1,024 ≈ 1,000, so for estimation we treat each "step" as ×1,000.
That lets you multiply by just adding exponents: 10⁸ items × 10³ bytes = 10¹¹ bytes = 100 GB.

| Power | Approx. | Name |
|---|---|---|
| 2¹⁰ | 10³ | KB (thousand) |
| 2²⁰ | 10⁶ | MB (million) |
| 2³⁰ | 10⁹ | GB (billion) |
| 2⁴⁰ | 10¹² | TB (trillion) |
| 2⁵⁰ | 10¹⁵ | PB (quadrillion) |

Use powers of **ten** for counts (users, requests), powers of **two** for bytes — and in practice convert the
bytes to powers of ten too, because the difference (~2–13%) is far below estimation precision.
*(GB vs GiB one-liner: 1 GiB = 2³⁰ ≈ 1.07×10⁹ B, ~7% more than 1 GB = 10⁹ B — irrelevant at ±2× precision; just
don't mix them within one calculation.)*

### 2.2 Time
- 1 day = 86,400 s ≈ **10⁵ s** (round up; you'll under-estimate QPS by ~14%, which is inside the noise)
- 1 month ≈ 2.5×10⁶ s (30 × 86,400 = 2.59×10⁶) · 1 year ≈ 3×10⁷ s (365 × 86,400 = 3.15×10⁷)
- ⭐ **Shortcut: 1 million/day ≈ 12 QPS** (1,000,000 / 86,400 ≈ 11.6). Scale it: 100M/day ≈ 1,200 QPS.

### 2.3 Jeff Dean latency numbers ("Latency Numbers Every Programmer Should Know")
Source: https://gist.github.com/jboner/2841832 (the gist labels them **~2012**; the original figures trace to
Jeff Dean / Peter Norvig around 2010). **Modern hardware is faster** — e.g. NVMe SSD random reads are now far
below 150 µs — but the **ratios between the tiers** are what you're memorizing, and those still hold in shape.

| Operation | Latency |
|---|---|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| Mutex lock/unlock | 25 ns |
| Main memory (RAM) reference | 100 ns |
| Compress 1 KB (Zippy — Google's internal name for Snappy) | 3 µs |
| Send 1 KB over 1 Gbps network | 10 µs |
| **Read 4 KB randomly from SSD** | **150 µs** |
| **Read 1 MB sequentially from memory** | **250 µs** |
| Datacenter round trip | 500 µs (0.5 ms) |
| Read 1 MB sequentially from SSD | ~1 ms |
| Disk (HDD) seek | 10 ms |
| Read 1 MB sequentially from HDD | 20 ms (older versions of the table: 30 ms) |
| Round trip CA ↔ Netherlands ↔ CA | 150 ms |

⭐ **Relative intuition beats rote — and you must say WHICH kind of access you mean:**
- **RANDOM access** (one small read at an unpredictable spot): RAM reference **~100 ns** vs SSD random read
  **~150 µs** → **~1,000×** (150,000 ns ÷ 100 ns = 1,500 — call it "three orders of magnitude"). HDD is far worse
  again: a seek alone is **10 ms**, ~100,000× RAM. *This is why caches exist:* a Redis lookup in RAM vs a database
  that has to hit disk for a random row.
- **SEQUENTIAL** (streaming 1 MB in order): memory **~250 µs** vs SSD **~1 ms** → only **~4×**. HDD 1 MB ≈ 20 ms →
  ~20× slower than SSD. *This is why sequential-write designs (logs, LSM trees, Kafka — Days 6/10/11) are fast
  even on disk:* streaming avoids the seek penalty.
- SSD vs HDD: **~20× sequential**, but **~65× random** (10 ms ÷ 150 µs) — random I/O is where HDDs die.
- **Same-DC round trip (0.5 ms) is cheap; cross-region WAN (~150 ms) dominates everything** → put data near users
  (CDN, Day 4), avoid chatty cross-region calls in a request path.

⭐ **Interview one-liner:** "Memory is roughly a thousand times faster than SSD for random reads but only ~4× for
sequential, and a cross-ocean round trip is 150 ms — so I cache hot random reads in RAM, write sequentially, and
keep the request path inside one region."

### 2.4 Data-size reference card
char/byte ~1 B · int64/long/timestamp ~8 B · UUID ~16 B · a tweet/short post ~**few hundred B – few KB** (incl.
metadata) · photo ~**200 KB–2 MB** · **1 hr HD video ~1 GB** (so 1 min ≈ ~15–20 MB).

---

## 3. Traffic inputs: DAU/MAU, stickiness, read/write ratio
- **DAU vs MAU** = Daily vs Monthly Active Users.
- **Stickiness = DAU / MAU.** 0.5 = very sticky (WhatsApp — people open it daily); 0.1 = occasional (a travel
  app). The interviewer often gives total users or MAU → you derive DAU via a **stated** stickiness.
  Example: "500M MAU, chat app, I'll assume 50% stickiness → 250M DAU."
- **Read:write ratio** — estimate reads and writes **separately** (e.g. 100:1 social feed). This one number decides
  the design: **read-heavy → caches + read replicas (Days 5/8); write-heavy → sharding + write-optimized stores
  (Days 9/10).** **Say it out loud** — it's the first "so what" you can give.

---

## 4. The five things you estimate — with the WHY behind each

### 4.1 QPS (and why peak is 2–10× average)
```
avg QPS  = (DAU × actions/user/day) / 86,400   (≈ /10⁵)
peak QPS = avg × (2 to 10)   ← state the factor + WHY
```
**Why average is a lie:** dividing by 86,400 pretends traffic is spread evenly across all 24 hours, including 4 a.m.
Real traffic follows a **diurnal (daily) curve** — people use apps while awake, with lunch and evening peaks.
- If most of a day's traffic lands in ~8 busy hours, the rate during those hours is about **24/8 = 3× the
  daily average**. That's where the common "×2–3" comes from.
- **Events** push it much higher: New Year's midnight greetings, a World Cup goal, a flash sale, a viral post —
  the Super Bowl/ticket-drop pattern can hit **×10 or more** for minutes.
- Global apps are smoother (one region's night is another's day) → lower factor; single-country apps peakier.

**Why it matters:** you **provision for peak, not average.** A system sized for the average falls over every
evening. So the number you divide by per-server capacity is always peak QPS.

Split into **read QPS vs write QPS** using the ratio.

⭐ One-liner: "Average is ~X; traffic is diurnal and event-driven, so I'll provision for ~3× peak, and ~10× if this
has flash-sale or live-event spikes."

### 4.2 Storage (and why replication multiplies it)
```
storage/day = items/day × bytes/item
total = storage/day × retention × replication   (+ metadata + indexes + growth headroom)
```
**Why replication multiplies:** from Day 8, we keep the **same data on several machines** to survive failures — a
common default is **3 copies**. Each copy is a full copy, so 90 TB of raw data occupies **270 TB of disks**. Forget
this and you under-buy by 3×. (Blob stores often use **erasure coding** instead, at ~1.2–1.5× overhead — say
"×3 for replicated DBs, less for erasure-coded blob storage" if you want the senior nuance.)

**Why add headroom:** indexes, metadata, and the fact that disks shouldn't run near 100% full — round up rather
than down. Project over **5 years** unless told otherwise; data grows, and the interviewer wants to see you size
for the future, not today.

### 4.3 Bandwidth (and why ×8)
```
bandwidth (bytes/s) = QPS × payload bytes
Mbps/Gbps = bytes/s × 8   ← networks are quoted in BITS
```
**Why ×8:** storage is measured in **bytes** (B), but network links and NICs are sold in **bits per second** (b/s).
1 byte = 8 bits. A "10 Gbps" NIC moves only 10/8 = **1.25 GB/s**. If you compute 60 MB/s and compare it to a
"1 Gbps" link without converting, you'll think you have 16× headroom when you really have ~2× (60 MB/s × 8 =
480 Mbps vs 1,000 Mbps).

⭐ The **×8** is the classic trap. *Reverse trap:* video bitrates (e.g. "5 Mbps stream") are **already in bits** —
don't multiply those by 8 again.

### 4.4 Memory / cache (and why the 80/20 rule)
~20% of data serves ~80% of requests → cache the hot 20%.
```
cache RAM ≈ (data you could serve) × 0.2
```
**Why 80/20 works:** popularity is **skewed** (a Pareto / Zipf-like distribution). Most tweets are seen by a few
people; a tiny number go viral and get millions of views. Today's links, this week's videos, celebrity profiles —
a small hot set takes most of the reads. So you don't need RAM for *all* data, just the hot head of the curve.
*(It's a rule of thumb, not a law — state it as an assumption.)*

**Hit-ratio math (why caches save databases):** at **90% hit ratio**, 100k read QPS → only **10k** reaches the DB
(the 10% misses). Raise it to 99% and the DB sees 1k. That one multiplication is often the justification for the
cache in your design.

### 4.5 Server count
```
servers = peak QPS / per-server capacity   (state the assumption, e.g. ~1,000 QPS/server for app logic)
```
Same logic for storage: `total ÷ usable disk per node = node count`, and for bandwidth: `peak bits/s ÷ NIC speed`.
**Why state the per-server number:** it varies 100× by workload (a cache lookup vs a heavy DB query). The method
is what's graded; say "assuming ~1k QPS per app server" and let the interviewer adjust it.

### Summary table (after the prose)
| Quantity | Formula | Don't forget |
|---|---|---|
| QPS | DAU × actions ÷ 10⁵ | peak ×2–10, split read/write |
| Storage | items/day × size × retention | × replication, 5-yr horizon |
| Bandwidth | QPS × payload | × 8 for bits |
| Cache | hot data × 0.2 | hit ratio → DB load |
| Servers | peak QPS ÷ per-server cap | state the cap |

---

## 5. Unit-conversion rules (memorize — this is where slips happen)
- **1 GB = 10⁹ B** → to go bytes→GB, **÷ 10⁹**.
- **1 TB = 10¹² B** → bytes→TB, **÷ 10¹²**.
- **1 PB = 10¹⁵ B = 1,000 TB.**
- ⭐ Quick anchor: **10¹² B = 10³ GB = 1 TB.**
- Bytes → bits (bandwidth): **× 8**.
- Carry exponents, don't write long zeros: 2×10¹⁰ × 10² = 2×10¹² (add exponents: 10 + 2 = 12).

---

## 6. Availability "nines" (reinforced from Day 1)
| Availability | Downtime/year |
|---|---|
| 99% | ~3.65 days |
| 99.9% | ~8.76 hours |
| 99.99% | ~52.6 minutes |
| 99.999% | ~5.26 minutes |

- Dependent services in **series multiply**: two 99.9% → 0.999² ≈ **99.8%** (worse than either). *Why:* the
  request fails if **either** fails.
- Redundant components in **parallel** improve it: two independent 99% replicas → 1 − (0.01 × 0.01) = **99.99%**.
  *Why:* the request fails only if **both** are down at once. This math justifies "add a replica."

---

## 7. Latency composition (how request latency adds up)
- **Sequential steps ADD** (A then B then C → A+B+C).
- **Parallel steps take the MAX** (fan out to 3 shards at once → ≈ slowest shard).
- Reason in **percentiles, not averages**: **p99** (slowest 1%) is what matters — at scale that 1% is millions of
  requests. In a fan-out, p99 is dominated by the *slowest branch* → "tail latency amplification." (If a page calls
  100 backends in parallel, each with a 1% chance of being slow, the chance the page is slow is 1 − 0.99¹⁰⁰ ≈ 63%.)

---

## 8. Worked examples (do these by hand until they're automatic)

### 8.1 URL shortener
100M new URLs/day · read:write = 100:1 · keep 5 years · ~500 B/row (short code, long URL, owner, timestamps).
1. **Write QPS:** 10⁸ / 10⁵ = 10³ = **1k/s avg** → peak (×3) **~3k/s**.
2. **Read QPS:** 100 × 1k = **100k/s avg** → peak **~300k/s**. *(Read-dominated → cache + replicas.)*
3. **Storage:** 500 B × 10⁸ = 5×10¹⁰ B = **50 GB/day** → × 365 × 5 = 50 × 1,825 ≈ 91,000 GB ≈ **~90 TB** →
   × 3 replicas ≈ **~270 TB** → doesn't fit one node → **shard**.
4. **Bandwidth (read):** 10⁵ × 500 B = 5×10⁷ B/s = 50 MB/s → × 8 = **400 Mbps** (one NIC — not a decision-maker).
5. **Cache:** assume the hot set is the recently created links (most clicks happen in a link's first days): even
   20% of a **week** of new links = 0.2 × 7 × 50 GB = **70 GB** → **tens of GB** → a small Redis cluster absorbs
   most of the 100k read QPS.
6. **So what:** read-dominated, ~270 TB → sharded KV store + Redis + replicas near users; writes trivial, reads &
   storage are the scaling axes.

### 8.2 Twitter-style (the classic interview question)
**Assumptions (say them):** 300M DAU · 2 tweets/user/day · read:write = 100:1 · ~1 KB per tweet for text +
metadata (ids 8 B each, timestamp 8 B, up to 280 chars which can be up to ~1 KB in UTF-8) · **10% of tweets
carry media, avg ~1 MB** · keep 5 years · 3× replication · peak factor ×3.

1. **Tweets/day:** 3×10⁸ × 2 = **6×10⁸**.
2. **Write QPS:** 6×10⁸ / 10⁵ = **6×10³ ≈ 6k/s avg** (exact ÷86,400 ≈ 6.9k — same order) → peak ×3 ≈ **~2×10⁴/s**.
3. **Read QPS:** 100 × 6k = **6×10⁵/s avg** → peak ≈ **~2×10⁶/s**. *(Strongly read-heavy → precomputed
   timelines/fan-out + heavy caching — Day 20.)*
4. **Text storage:** 6×10⁸ × 10³ B = 6×10¹¹ B = **600 GB/day** → × 1,825 days ≈ 1.1×10¹⁵ B ≈ **~1.1 PB** →
   × 3 ≈ **~3.3 PB**.
5. **Media storage:** 10% of 6×10⁸ = 6×10⁷ media tweets × 10⁶ B = 6×10¹³ B = **60 TB/day** → × 1,825 ≈
   1.1×10¹⁷ B ≈ **~110 PB** → × 3 ≈ **~330 PB** (or ~1.5× with erasure coding ≈ ~165 PB).
   → **Media is ~100× the text.** That's the insight.
6. **Servers (read path):** peak 2×10⁶ QPS ÷ assumed ~10⁴ QPS per cache-backed timeline server = **~200 servers**;
   at a more conservative ~10³ QPS/server it'd be ~2,000. *(Show the assumption — the method is what's scored.)*
7. **So what:** text fits a sharded DB (single-digit PB); **media dominates storage → object/blob store (S3-like)
   + CDN**; the read path (2M QPS peak) — not writes — drives caching and timeline fan-out design.

### 8.3 Photo + video storage and peak bandwidth (Instagram / YouTube style)

**(a) Instagram-style photo storage, 5 years.** Assume 100M photo uploads/day, ~2 MB each (original + resized
copies rounded together).
- Per day: 10⁸ × 2×10⁶ B = 2×10¹⁴ B = **200 TB/day**.
- 5 years: 200 TB × 1,825 ≈ 3.65×10⁵ TB = **~365 PB**; × 3 replication ≈ **~1.1 EB** (exabyte = 1,000 PB).
- **So what:** this is blob storage, not a database → object store with hot/cold tiers (old photos move to
  cheaper cold storage); only the photo **metadata** (id, owner, URL, timestamp — a few hundred bytes) goes in a DB.

**(b) YouTube-style peak streaming bandwidth.** Assume 100M DAU · each watches ~30 min/day · average stream
~5 Mbps (HD) · peak factor ×3.
1. **Total watch-seconds/day:** 10⁸ × 1,800 s = 1.8×10¹¹ s.
2. **Average concurrent viewers:** 1.8×10¹¹ / 86,400 ≈ **~2×10⁶** (at any instant, ~2M people are watching).
3. **Peak concurrent:** × 3 ≈ **6×10⁶**.
4. **Peak egress:** 6×10⁶ × 5×10⁶ bits/s = 3×10¹³ b/s = **~30 Tbps**. *(Bitrate is already in bits — no ×8.)*
5. **Servers:** at an assumed ~10 Gbps usable per edge server → 3×10¹³ / 10¹⁰ = **~3,000 edge servers**.
6. **Upload storage (bonus):** if 1% of DAU uploads one ~5-min video/day → 10⁶ videos × ~100 MB (1 hr HD ≈ 1 GB,
   so 5 min ≈ 80 MB, rounded up) = 10¹⁴ B = **100 TB/day raw**; × ~3 renditions (360p/720p/1080p…) ≈ 300 TB/day;
   × 3 replication ≈ **~1 PB/day**.
7. **So what:** 30 Tbps can't come from one origin → **CDN is mandatory** (Day 4), with popular videos cached at
   the edge (80/20 again); storage grows ~1 PB/day → tiered blob storage; transcoding is an async pipeline (Day 11).

### 8.4 ⭐ Rate-limiter memory estimate (Day 14 cross-ref — "estimate memory for N million users")
**Assumptions:** 10M active users, one token bucket per user stored in Redis as a small hash.
1. **Per-key bytes (state each):**
   - key name, e.g. `rl:{user_8472910}:api` → **~25–30 B**
   - token-bucket state = **2 numbers** (`tokens`, `ts`) + their field names → **~40–50 B**
   - **Redis per-key bookkeeping** (hash-table entry, object headers, the TTL entry because we set `EXPIRE`) →
     *roughly* **~50–100 B** — this is an approximation; it varies by Redis version and encoding (verify with
     `MEMORY USAGE <key>` on real data).
   - **Total ≈ ~150–200 B/key** → round to **200 B**.
2. **Total:** 10⁷ × 200 B = 2×10⁹ B = **~2 GB**; with one replica for failover × 2 ≈ **~4 GB**.
3. **Contrast — sliding-window LOG:** if each user may have up to 100 timestamps in the window, stored in a sorted
   set at very roughly ~50 B per entry: 10⁷ × 100 × 50 B = 5×10¹⁰ B = **~50 GB** — **~25× more**. That's the
   number that justifies "sliding-window counter or token bucket, not the log."
4. **So what:** memory is **not** the bottleneck — 2–4 GB fits one Redis node. The real constraint is **throughput**:
   if the API does 10⁶ requests/s and every request does one limiter check, that's 10⁶ Redis ops/s — beyond a
   single Redis node's order-of-10⁵ simple ops/s → **shard the limiter keys across a Redis Cluster** (and/or use a
   local pre-limiter, Day 14 §6). *This is estimation doing its job: it moved the worry from memory to QPS.*

---

## 9. ⚠️ Common pitfalls (the ones that cost points)
1. **Bytes vs bits** — forgetting ×8 for network bandwidth (or wrongly ×8-ing a bitrate already in bits).
2. **GB vs GiB** — ~7% difference; irrelevant for estimation, just stay consistent.
3. **Forgetting replication** — raw bytes ≠ disk bought; ×3 for replicated stores.
4. **Forgetting peak** — sizing servers for the average; always multiply and say why.
5. **Unit slips of 1000×** — bytes→GB is ÷10⁹, bytes→TB is ÷10¹² (my Task 2 slip). Sanity-check the magnitude.
6. **False precision** — "6,944.4 QPS" signals you don't understand estimation; say "~7k".
7. **Estimating things that don't change a decision** — wastes time; skip or one-line them.
8. **No "so what"** — numbers without the conclusion ("→ so storage forces sharding") earn little.

---

## My practice — Task 2, WhatsApp-lite (graded 8.5/10)
**Inputs:** 1B users · 500M DAU · 40 msg/day · 100 B/msg · 30-day retention.
1. **Messages/day** = 500M × 40 = 2×10¹⁰
2. **avg write QPS** = 2×10¹⁰ / 10⁵ = **2×10⁵** → **peak (×3) = 6×10⁵**
3. **Storage/day** = 2×10¹⁰ × 100 B = 2×10¹² B = **2 TB/day** → **30-day = 60 TB**
   - ⚠️ my slip: wrote 2×10⁶ GB (1000× off) — forgot the ÷10⁹ bytes→GB. Correct = 2×10³ GB = 2 TB.
   - *(Audit note: 60 TB is raw; with 3× replication it's ~180 TB on disk — still small. Using ÷86,400 instead of
     ÷10⁵ gives ~2.3×10⁵ avg QPS — same order, the rounding is fine.)*
4. **Peak write bandwidth** = 6×10⁵ × 100 B = 6×10⁷ B/s = 60 MB/s → **×8 = 480 Mbps**
5. **So what:** **write-heavy**; storage (60 TB) is small — real problem is **high write QPS + real-time fan-out** to recipients → append-optimized store (Cassandra) + message queue + persistent WebSocket connections.

*(Audit: every arithmetic step above re-checked — all correct.)*

**Lessons for me:**
- ⭐ **bytes→GB = ÷10⁹; bytes→TB = ÷10¹².** (10¹² B = 1 TB.)
- ⭐ Answer **every sub-part** — I skipped bandwidth + "so what" (my #1 recurring weak area). Every box is scored.

---

## Delivery tips
State assumptions · round every step · carry exponents · **always end with the "so what"** (the bottleneck) · sanity-check magnitude · say "roughly / order-of / ±2×" — method + confidence beats false precision · **don't estimate what won't change a decision.**

➡️ **Next — Day 16 (answer framework):** estimation slots in **after requirements** and **only where it changes a
decision**; the framework tells you when to do it and how long to spend.

---

## Personal Notes (recurring gaps)
- [ ] ⭐ **bytes → GB is ÷10⁹** (and → TB is ÷10¹²). My Task 2 slip wrote 2×10⁶ GB instead of 2×10³ GB — a 1000×
  error. Fix: write the exponent subtraction explicitly (10¹² ÷ 10⁹ = 10³) and sanity-check "does 2 PB/day of
  chat text make sense?" (No.)
- [ ] ⭐ **Answer every sub-part** — I skipped the bandwidth line and the "so what" until prompted. Same #1 gap as
  Days 8, 9, 13, 14. On an estimation question every line (QPS, peak, storage, bandwidth, cache, servers, so-what)
  is its own scored box.
- [x] **QPS chain clean** ✅ — DAU → messages/day → avg QPS → peak flowed correctly with powers of ten.
