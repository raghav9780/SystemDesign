# Day 1 — Scalability: Vertical vs Horizontal Scaling
**Phase 1: Foundations** | Score: 7/10

> **One-sentence idea:** Scalability = your system's ability to keep working as the load grows
> instead of buckling under it. You get there in one of two ways — **make the one machine bigger**
> (vertical) or **add more machines and split the work** (horizontal) — and almost every hard part
> of system design comes from the second choice, because the moment you have many machines you must
> answer: *where does the data live, and what happens when one machine dies?*

## Core Concept — what "scalability" actually means

Scalability is a system's ability to **handle growth without falling over.** Growth means more users,
more requests per second, more data. A scalable system absorbs that and keeps responding; an
unscalable one slows to a crawl or crashes.

A useful mental picture: a single coffee shop with one barista. When two customers walk in, fine.
When two hundred walk in at once, the line stops moving. "Scaling" is everything you do so the line
keeps moving — and you have exactly two levers to pull, below.

---

## Two Ways to Scale

### 1. Vertical Scaling (Scale UP ⬆️) — buy a bigger machine
This means giving your **one** server more power: more CPU, more RAM, a faster disk. Same single box,
just beefier. Like replacing your one barista with a superhuman barista who works ten times faster.

- ✅ **Simple** — no code changes and none of the headaches that come from coordinating many machines.
  You just resize the server and carry on.
- ❌ **Hard ceiling** — a machine can only get so big. There is a most-powerful server money can buy,
  and once you hit it, you're stuck.
- ❌ **Single Point of Failure (SPOF)** — a SPOF is any one component whose death takes the *whole*
  system down with it. With one box, that box is your SPOF: it dies, everything dies.
- ❌ **Disproportionately (super-linearly) expensive at the high end** — the last bit of power costs wildly more than the
  first. Doubling a small server is cheap; doubling a top-tier server costs a fortune.

### 2. Horizontal Scaling (Scale OUT ➡️) — add more machines
Instead of one super-barista, you hire **many ordinary baristas** and split the customers among them.
You add more servers and spread the work across all of them.

- ✅ **Near-infinite scaling** — out of capacity? Add another node. And another. There's no single
  ceiling the way there is with one big box.
- ✅ **Enables fault tolerance** — because no single machine is doing everything, one node dying can
  mean the others pick up its share and the system survives. But note: horizontal scaling *enables*
  fault tolerance, it doesn't *guarantee* it. You still need redundancy (real spare copies), health
  checks (to detect the dead node), and spare capacity (headroom to absorb its load). Without those,
  the survivors just inherit the dead node's traffic, get overwhelmed, and fall over one by one — a
  **cascading failure**.
- ❌ **Complex** — now you need a **load balancer** (a traffic director that decides which server each
  request goes to → Day 2), you have to keep **data consistent** across machines, and you have to
  figure out **where state lives** (more on that below). This complexity is the price of admission.

---

## Key Mental Model

The one-liner to remember: **Vertical = easy but limited. Horizontal = hard but unlimited.**

In the real world you don't pick one forever — you sequence them:

- Real systems **start vertical** because it's cheap and simple, then **go horizontal** once they've
  outgrown what a single machine can do.
- ⭐ **Scale vertically FIRST.** It's an instant band-aid — bumping the server size buys you time
  *today* with zero re-architecting. Meanwhile you **plan the horizontal move in parallel**, so you're
  ready before the vertical ceiling hits.
- ⭐ **The hard part of horizontal scaling is STATE.** "State" means the data that has to be remembered
  between requests — who's logged in, what's in their cart, their session. With one machine, state just
  lives there. With many machines the question becomes: *which machine remembers it?* — and getting that
  wrong is the source of most horizontal-scaling pain (see the State trap below).

---

## Traps & Gotchas

These are the two classic ways a "scaled" system is secretly still broken.

- **DB trap — you moved the SPOF, you didn't remove it.** Say you add five web servers but point them
  all at **one** database. You've scaled the web tier, but every request still funnels into that single
  DB. You haven't eliminated the single point of failure — you've just **relocated** it from the web
  tier to the data tier. The web layer is scaled; the data layer is still the bottleneck and still the
  thing whose death kills everything.

- **State trap — the "why am I logged out?" bug.** A user logs in, and that login session is stored on
  **Server A**. Their next request gets routed to **Server B**, which never heard of them → they appear
  logged out. The fix is to make the web tier **stateless**: the servers themselves remember nothing
  between requests, and the session/user data is pushed out to a **shared store** (e.g. a shared cache
  or database) that *every* server can read. Now it doesn't matter which server you land on. This is the
  concrete payoff of solving the "where does state live?" question above.

> **Stateless vs Stateful** (defining the terms, since they drive everything here): a **stateful** server
> keeps important data on itself between requests (like Server A holding your session). A **stateless**
> server keeps nothing locally — every request carries or fetches what it needs from a shared store.
> Stateless is what makes horizontal scaling work, because any server can handle any request.

---

## Solving the DB Bottleneck (order of effort)

Once the database is your bottleneck (the DB trap above), you fix it in increasing order of effort.
Don't jump to the hard option first — climb the ladder.

1. **Cache** (Redis / Memcached) in front of the DB. A cache is a small, fast store that holds
   frequently-requested answers so you don't hit the DB every time. In **read-heavy** systems (many
   are — but confirm, don't assume), reads dominate: people view data far more than they change it, so
   caching the hot reads takes huge load off the DB cheaply. Watch out though — **write-heavy** systems
   (analytics ingestion, IoT sensor streams, logging pipelines, chat) flip this, and a read cache buys
   you little there; you reach for sharding and write-optimized stores instead. → Day 3
2. **Read replicas** — keep extra copies of the database. **Reads go to the replicas, writes go to the
   primary** (the one authoritative copy). This spreads out read load without touching write logic. → Day 8
3. **Shard** — split the data itself across multiple databases, each owning a slice (e.g. users with
   names A–M on one DB, N–Z on another). Now no single DB holds everything, so writes *and* reads
   spread out — but this is the most complex step, which is why it's last. → Day 9

⭐ **Memorize the ladder: Cache → Replicate → Shard.** Cheapest/easiest first, hardest last.

---

## Storing Files (photos/videos)

When your app handles big files like images and videos, there's a specific right answer.

- ❌ **DON'T store large blobs in the database.** A "blob" (Binary Large OBject) is a big chunk of binary
  data like a photo or video file. Stuffing these into the DB bloats it, slows every query, and makes
  backups painful — the database is built for structured records, not heavy media.
- ✅ **Store files in object storage** (e.g. AWS S3), and keep only the **URL/path** to the file in the
  database. **Object storage** is a service purpose-built to hold huge numbers of files ("objects"),
  each retrievable by a key/URL — exactly the job a database is bad at.
- **Why this matters for scaling:** because the file lives in one central shared store, **any** server
  can serve it just by handing out the URL. That keeps the web tier **stateless** (the same property
  from the State trap) — no server is special because no server holds the file locally.
- **Serve via CDN for geographic speed.** A **CDN** (Content Delivery Network) is a network of servers
  spread around the world that cache your files close to users, so someone in India isn't waiting on a
  file that lives in the US. → Day 4

---

## Vocabulary to Remember

A quick reference for the terms above (each is defined in full where it first appears):

- **SPOF** = Single Point of Failure — one component whose death takes the whole system down.
- **Stateless vs Stateful** — stateless servers remember nothing locally; stateful ones hold data
  between requests. Stateless is what enables horizontal scaling.
- **Object Storage (S3)** — a service for storing large files by key/URL, instead of in the DB.
- **Load Balancer** — the tool that routes incoming traffic across your servers. → Day 2

---

## ⭐ Interview-completeness additions (audit pass)

These are the concepts an interviewer expects you to reach for once you've established *how* to scale.
They turn "add more machines" into decisions you can defend with numbers.

- **Availability "nines" (and the math that justifies every replica).** Availability is the fraction of
  time your system is actually up, and it's quoted in "nines." The intuition: each extra nine is an
  order of magnitude less downtime, and it gets expensive fast. **99%** = ~3.65 days down per year.
  **99.9%** ≈ 8.76 hours. **99.99%** ≈ 52.6 minutes. **99.999%** ("five nines") ≈ 5.26 minutes. Now the
  part that actually drives design: **services chained in SERIES multiply, and the product is worse than
  any single link.** If a request must pass through two components that are each 99.9% available, the
  path is 0.999 × 0.999 ≈ **99.8%** — you *lost* availability by adding a dependency. But **redundant
  components in PARALLEL** flip it: if either of two 99%-available replicas can serve the request, the
  pair is 1 − (0.01 × 0.01) = **99.99%**. *That* is why "add a replica" or "add a second load balancer"
  is the reflexive answer — parallel redundancy is how you buy nines.

- **Latency vs throughput, tail latency, and performance vs scalability.** Keep two axes separate:
  **latency** is how long one request takes; **throughput** is how many requests you handle per second.
  And don't confuse **performance** (how fast a single request is) with **scalability** (does that speed
  *hold* as load grows) — a system can be fast at 10 users and collapse at 10,000. The pro move is to
  **reason in percentiles, not averages.** p50 (median) hides your worst experiences; **p95/p99** are the
  slow tail. Why p99 matters: at scale, 1% of a million requests is 10,000 unhappy users every cycle,
  and a single user often triggers many backend calls, so they *hit* the tail constantly. Averages lie;
  percentiles tell the truth.

- **Redundancy & failover patterns (the "how" behind removing a SPOF).** Removing a single point of
  failure means having a backup ready and a way to switch to it. **Active-passive:** one node serves, an
  idle standby waits — cheaper (you're paying for an idle box) but failover isn't instant (the standby
  must detect the death and take over). **Active-active:** all nodes serve traffic simultaneously — no
  wasted capacity and instant tolerance, but now you need a load balancer in front and you must
  coordinate shared state across them. **N+1 provisioning** is the sizing rule: run one more node than
  your peak load actually needs, so losing one still leaves enough. And none of this works without
  **health checks** — the automated "are you alive?" pings that *detect* a failure so failover can fire.

- **Where does state go once servers are stateless? (externalization options).** Making the web tier
  stateless (from the State trap) just moves the question: where does the session live now? Three real
  answers. **(1) A shared session store** like Redis — fast, central, every server reads it. **(2) The
  database** — simpler, durable, but heavier per request. **(3) A signed token (JWT)** — the server
  stores *nothing*; the client carries a cryptographically signed token that *is* its own proof of
  login, and any server can verify it without a lookup. The **anti-pattern** to name and reject:
  **sticky sessions** (pinning a user to the one server that has their session) — it re-introduces the
  statefulness you just worked to remove and breaks the moment that server dies.

- **Read/write ratio as a scaling lens — say it out loud.** Before choosing tools, estimate the ratio.
  A **100:1 read-heavy** workload points you straight at **caches + read replicas** (cheap wins on the
  read path). A **write-heavy** workload means caches and replicas won't save you — writes still all
  land on the primary — so you reach for **sharding and write-optimized stores** much sooner. Stating
  the ratio explicitly is what makes your scaling choice sound deliberate instead of guessed.

- **Autoscaling / elasticity.** **Autoscaling** is automatically *adding* servers when a metric climbs
  (CPU, requests-per-second, or queue depth) and *removing* them when it drops — you pay for capacity
  only when you need it. The reason it works at all is the **stateless** tier: if any server can handle
  any request, you can spin them up and kill them freely. The catch — and it ties the whole day together
  — is that **the database is the thing autoscaling can't easily grow.** You can't just clone a stateful
  primary the way you clone a web server, which is *exactly* why the Cache → Replicate → Shard ladder
  exists: it's the manual, deliberate work of scaling the one tier elasticity can't.

- **Object storage, one level deeper.** Three storage shapes worth distinguishing: **block storage** (a
  raw virtual disk you attach to a server — fast, low-level), **file storage** (a shared filesystem with
  folders and paths), and **object storage** (S3 — files addressed by a key/URL, built for massive
  scale). Two details interviewers probe: **presigned URLs** — instead of routing an upload *through*
  your app servers, you hand the client a temporary signed URL and it uploads **directly to S3**, so the
  heavy bytes never touch your servers; and **storage tiers** — **hot** (accessed constantly, priciest),
  **warm**, **cold**, and **archive** (rarely touched, cheap, slow to retrieve) — matching cost to how
  often data is actually read.

- **One-line CAP preview.** The moment your data lives on many machines, a network hiccup can split them
  apart (a *partition*), and during that split you can't have both perfect **consistency** (everyone sees
  the same data) *and* full **availability** (everyone still gets an answer) — you must trade one off.
  That's **CAP**, and it's the whole story of → Day 7.

---

## ⭐ Completeness additions (pass 2)

### 1. SLI vs SLO vs SLA — three words for "how reliable are we?"
The nines above tell you *how much* downtime a percentage means. These three terms tell you *who measures
what, and who gets in trouble*. Think of a pizza shop promising delivery times:
- **SLI — Service Level *Indicator*** = the **measurement** itself, a number you actually observe. Pizza
  example: "% of pizzas delivered within 30 min." System example: **"% of HTTP requests in the last 30 days that
  returned a non-5xx response in under 300 ms."**
- **SLO — Service Level *Objective*** = the **internal target** for that SLI. "We aim for 99.9% of requests
  to succeed under 300 ms over 30 days." It's what your engineering team designs and alerts against. The gap
  between 100% and the SLO is your **error budget** (99.9% → 0.1% of requests may fail) — spend it on risky
  deploys; when it's used up, slow down and fix reliability.
- **SLA — Service Level *Agreement*** = the **external promise with consequences**, usually in a contract:
  "If monthly availability drops below 99.5%, the customer gets a 10% credit." Note it's normally set
  **looser than the SLO**, so you get warned (SLO breached) *before* you owe anyone money (SLA breached).

Memory hook: **SLI = what you measure, SLO = what you aim for, SLA = what you promise (and pay for).**
Interview line: *"I'd define an availability SLI and latency SLI (p99), set SLOs like 99.9% / p99 < 300 ms, and
keep the customer SLA looser than the SLO."* (Full observability — metrics, logs, traces, alerting on SLOs —
is Day 22.)

### 2. The AKF Scale Cube — three directions to scale
A handy picture for "how else can I scale?" — three axes, each a different way to split the work:
- **X-axis = clone.** Run **N identical copies** of the same thing behind a load balancer. This is plain
  horizontal scaling of a stateless tier (and read replicas are X-axis for the DB). Easiest; doesn't help if the
  data itself is too big.
- **Y-axis = split by function.** Break the app into **different services by job**: a login service, a
  payments service, a search service — each scaled on its own. This is the microservices direction.
- **Z-axis = split by data.** Every server runs the same code but owns **a different slice of the data** —
  users A–M here, N–Z there. This is **sharding** (Day 9).

Real systems combine all three (e.g. a separate payments service [Y], sharded by user [Z], with several copies
of each shard's app servers [X]). Naming the cube is a nice way to show you know there are more scaling
dimensions than "add servers."

### 3. Limits of vertical scaling — one more cost: resize downtime
On top of the ceiling, the SPOF, and the price curve: making one box bigger usually means **stopping it to move
it to a bigger instance type** (on most clouds, changing the instance size requires a stop/restart). So even the
"easy" option often costs a **maintenance window** — another reason not to rely on it forever.

### 4. Automated vs manual failover
Once you have a standby (active-passive above), *who* flips the switch?
- **Manual failover** — a human gets paged, confirms the primary is really dead, and promotes the standby.
  Safe from false alarms, but slow: minutes to tens of minutes of downtime, at 3 a.m.
- **Automated failover** — health checks detect the failure and the system promotes the standby by itself in
  seconds. Fast, but dangerous if detection is wrong: a primary that's just *slow* might get declared dead
  while it's still running → two primaries both accepting writes (**split-brain**, covered on Day 8).
Most production systems automate failover for the stateless tier and use automated-but-carefully-guarded
failover (quorums, fencing) for databases.

### 5. Object storage — the last two details: flat namespace + metadata, and CDN pairing
- **Flat namespace + metadata.** Object storage has **no real folders**. A key like `users/42/avatar.jpg`
  *looks* like a path, but it's just one long name in a giant flat key→object map; the "folders" are a display
  trick. Each object carries **metadata** (content type, size, custom tags). The flat design is *why* it scales
  to billions of objects — there's no directory tree to lock or rebalance.
- **Presigned URLs work for downloads too** — a short-lived signed link lets a user fetch a *private* file
  directly from storage without making it public.
- **Pair it with a CDN:** object storage is the **origin**, the CDN caches popular files near users (Day 4).
  The full pattern: client uploads via presigned URL → object store → DB keeps only the key/URL + metadata →
  reads served through the CDN.

### 6. "Scale from 1 user to millions" — the whole day as one story
This is a classic opening question; tell it as a sequence, each step fixing the bottleneck the last one exposed:
1. **One server** runs web app + DB. Fine for a demo.
2. **Separate the DB** onto its own machine — the two compete for resources less, and each can be sized on its own.
3. **Vertical scale** both while it's cheap.
4. **Load balancer + several stateless web servers** (sessions moved to Redis/JWT) — web-tier SPOF gone.
5. **Cache** hot reads (Day 3) and put static files in **object storage + CDN** (Day 4).
6. **Read replicas** for the DB (Day 8); automated failover for the primary.
7. **Async work via a message queue** (sending emails, resizing images — Day 11) so slow jobs don't block requests.
8. **Shard** the DB when writes or data size outgrow one primary (Day 9).
9. **Multiple data centers/regions** with geo-routing (Day 4) for latency and disaster survival.
Each step: *name the bottleneck, then the fix* — that's the "finish the thought" habit from the Personal Notes below.

---

## Personal Notes (gaps to fix)
- [ ] **Finish thoughts:** don't just DIAGNOSE ("X is broken") — PRESCRIBE ("so I'd do Y").
- [ ] **Use proper NAMES for concepts** (e.g. "object storage / S3", not "upload to AWS").
