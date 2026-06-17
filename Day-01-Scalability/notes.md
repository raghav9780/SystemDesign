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
- ❌ **Exponentially expensive at the high end** — the last bit of power costs wildly more than the
  first. Doubling a small server is cheap; doubling a top-tier server costs a fortune.

### 2. Horizontal Scaling (Scale OUT ➡️) — add more machines
Instead of one super-barista, you hire **many ordinary baristas** and split the customers among them.
You add more servers and spread the work across all of them.

- ✅ **Near-infinite scaling** — out of capacity? Add another node. And another. There's no single
  ceiling the way there is with one big box.
- ✅ **Fault tolerant** — because no single machine is doing everything, one node dying just means the
  others pick up its share. The system survives.
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
   frequently-requested answers so you don't hit the DB every time. Since **most traffic is reads**
   (people viewing data far more than changing it), caching the hot reads takes huge load off the DB
   cheaply. → Day 3
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

## Personal Notes (gaps to fix)
- [ ] **Finish thoughts:** don't just DIAGNOSE ("X is broken") — PRESCRIBE ("so I'd do Y").
- [ ] **Use proper NAMES for concepts** (e.g. "object storage / S3", not "upload to AWS").
