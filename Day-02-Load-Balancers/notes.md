# Day 2 — Load Balancers
**Phase 1: Foundations** | Score: 8/10

> **One-sentence idea:** A load balancer is the **traffic cop standing in front of your server fleet** —
> every request hits it first, and it decides which server handles it, quietly skipping any server that
> has died. Think of a **restaurant host** who seats arriving guests across many tables so no single
> waiter gets swamped, and who stops seating guests at a table whose waiter just went home.

The single biggest lesson of the day is at the very bottom and it's worth saying up front: **state must
live in a SHARED store, never on the individual server.** Almost every "good fix" today is really that
idea in disguise.

---

## What a Load Balancer Does

Picture every user request as a customer walking through one front door. Behind that door you don't have
one worker — you have a *fleet* of identical servers. Something has to stand at the door and point each
customer at a worker. That something is the **load balancer (LB)**: it sits between the users and the
server fleet, and **every request hits it first**.

It does three jobs:
1. **Distribute traffic** — spread incoming requests across the fleet so no single server is overwhelmed.
2. **Health checks** — quietly ping each server; if one stops answering, **drop it from rotation** so no
   customer gets sent to a dead worker. When it recovers, put it back.
3. **Hide the fleet** — the outside world sees **one address** (the LB), not the messy list of individual
   server IPs. You can add or remove servers behind the scenes and nobody outside notices.

⭐ Why it matters: on Day 1 the web tier was a **SPOF** (single point of failure — one box whose death
takes the whole app down). Putting an LB in front of *multiple* web servers removes that web-tier SPOF,
because any one server can die and the LB just routes around it.

---

## L4 vs L7 — how deeply does the LB *look* at a request?

A request arriving over the network has layers, like an envelope inside an envelope. The LB can choose to
read only the *outer* envelope (fast, shallow) or open it up and read the *letter inside* (slower, smart).
Those two choices are named after layers of the networking stack.

### L4 (Transport layer) — the shallow, fast reader
L4 routes using only the **IP address and the TCP/UDP port** — the outer envelope. (TCP and UDP are the
two transport protocols that move data; the "port" is just the numbered door on the server, e.g. 443 for
HTTPS.) It never opens the envelope to read *what* the request is actually asking for.
- ✅ **Very fast, low overhead** — there's barely anything to inspect, so it just forwards.
- ❌ **Dumb** — because it can't see the content, it can't make any content-based decisions.

### L7 (Application layer) — the smart, content-aware reader — *most modern web apps*
L7 opens the envelope and reads the actual **content of the request**: the **URL path**, the **headers**
(metadata the browser attaches), and **cookies** (small tags the browser carries to identify a session).
Because it understands the request, it can do clever things:
- **Path-based routing** — send `/video/*` requests to the video servers and `/api/*` to the API
  servers. (One front door, but customers get routed by what they came for.)
- **SSL termination** — the LB handles the encryption/decryption of HTTPS so the back-end servers don't
  have to. ("SSL" is the encryption layer behind HTTPS; "terminating" it means the secure connection
  ends *at the LB*.)
- **Caching** — keep a copy of common responses so it can answer without bothering a server.
- **Compression** — shrink responses before sending them out.

Trade-off:
- ✅ **Smart** — content-aware routing and the features above.
- ❌ **More CPU → slightly slower**, because actually reading and understanding each request costs work.
- Real examples: **Nginx**, **AWS ALB** (Application Load Balancer).

---

## Routing Algorithms — *how* the LB picks a server

Once the LB decides to forward a request, it still has to choose *which* server. Different algorithms suit
different situations.

- **Round Robin** — hand requests out in a fixed cycle: server 1, 2, 3, 1, 2, 3… Like dealing cards
  evenly around a table. Simple, but it **assumes every server and every request is equal** — which often
  isn't true.
- **Weighted Round Robin** — give each server a **weight** and send the beefier servers more traffic. A
  machine with twice the CPU gets twice the share. Use it when your fleet is a mix of strong and weak boxes.
- **Least Connections** — send the next request to whichever server currently has the **fewest active
  connections**. ⭐ **Best when requests vary wildly in duration** — e.g. heavy long-lived video streams
  vs light, instant API calls. Round robin would blindly pile a new heavy stream onto a server already
  juggling three; least-connections naturally steers toward the server that's actually free.
- **Least Response Time** — send to whichever server is **answering fastest right now**. A step beyond
  least-connections: it reacts to real measured speed, not just connection count.
- **IP Hash** — take the user's IP address, run it through a hash function (a math function that turns
  the IP into a number), and use that to **always pick the same server for that user**. This is one way
  to "stick" a user to a server — which leads directly into the next topic.

---

## Sticky Sessions (Session Affinity) 🍯

A **session** is the server-side memory of who you are and what you're doing — your logged-in state, your
shopping cart, etc. **Sticky sessions** (a.k.a. **session affinity**) means the LB **pins a user to the
SAME server for their whole session**, usually via a cookie the LB sets or via IP hash (above). The reason
people reach for this: if the server is keeping *your* session in its own local memory, then you'd better
keep coming back to *that* server, or your cart vanishes.

- ✅ **Easy fix for stateful apps** — if a server stores session data locally, stickiness keeps each user
  returning to the server that holds their data, so nothing breaks.
- ❌ **But it breaks the whole point of load balancing.** Two problems:
  - If that one server **dies, all its users lose their sessions** (their cart, their login) — the LB can
    route them elsewhere, but the new server has no memory of them.
  - Traffic gets **uneven**: users are glued to specific servers, so you can't freely rebalance load.
- ✅✅ **The BETTER fix — make servers stateless.** Don't store the session on the server at all. Put it in
  a **shared store like Redis** (a fast in-memory data store all servers can reach). Now **any server can
  serve any user**, because the session lives in one shared place rather than in one server's head. Once
  you do this, **sticky sessions become unnecessary** — you can go back to plain round-robin / least-
  connections and lose nothing when a server dies.

⭐ This is the signature lesson appearing for the first time: **the state moved off the server into a shared
store**, and that single move dissolved the problem.

---

## SPOF Ladder (KEY LESSON)

Here's a trap that catches beginners. You add a component to *fix* a single point of failure — but that
new component can quietly **become** a single point of failure itself. So you have to keep climbing the
ladder, asking at each rung: *"and is THIS thing now a SPOF?"*

- You had multiple **web servers** to remove the web-tier SPOF… but now **the LB itself is a SPOF** (if
  the one LB dies, nothing reaches any server). → Fix: run **multiple LBs** in active-active or
  active-passive mode with **failover**. ("Active-active" = both work at once; "active-passive" = a
  standby waits to take over; "failover" = the automatic hand-off when one dies.)
- You added a **Redis session store** to make servers stateless… but now **Redis is a SPOF** (if it dies,
  every server loses access to all sessions). → Fix: **Redis replication + clustering + failover** (keep
  copies of Redis running so one can take over).

⭐ The rule: **the fix for a SPOF is to ADD REDUNDANCY, never to delete the component.** You don't remove
the LB or Redis — you make sure there's always more than one of it.

---

## Large File Uploads (e.g. a 2GB video, and the server might be killed mid-transfer)

Big uploads expose a nasty edge case: what if the upload is 90% done and the server handling it gets
killed (deploy, crash, scale-down)? Starting a 2GB upload over from zero is miserable. Here's the toolkit.

- **Chunking / multipart upload** — split the file into app-level **chunks** of ~5–10MB and upload each
  one **independently**. If one chunk fails, you only re-send that chunk, not the whole file.
  - ⚠️ Naming precision: **"chunks" = the app-level pieces YOU define**; **"packets" = the tiny TCP/OS-level
    pieces the network breaks data into automatically.** They are different layers — don't say "packets"
    when you mean "chunks."
- **Resumable upload** — keep track of the **last acknowledged chunk**, so after a failure you **resume at
  chunk N+1** instead of restarting. (This is what makes a dropped upload recoverable.)
- **CRITICAL — chunks must go to shared object storage (S3), NOT the server's local disk.** "Object
  storage" (e.g. Amazon S3) is a shared store for files that *all* servers can reach. If chunks piled up
  on one web server's local disk and that server died, the half-finished upload would be stranded there.
  Sending chunks to **S3** instead means S3 **tracks the received chunks centrally**, so **any server can
  resume any upload** → the web tier stays **stateless** (same lesson as sessions: state lives in the
  shared store).
- **Pro move — pre-signed URLs.** A **pre-signed URL** is a temporary, permission-granting link the server
  hands to the client. The client then **uploads chunks straight to S3 using that link**, completely
  **bypassing the web servers for the heavy data transfer**. The servers' only jobs are to **give
  permission** (issue the URL) and **record metadata** (note that the file exists). The gigabytes never
  flow through your app servers at all.

---

## Vocabulary
A quick checklist of the day's terms — each is defined inline above:
- **L4 / L7**, **SSL termination**, **path-based routing**
- **Session affinity / sticky sessions**
- **Least Connections**, **IP Hash**
- **Multipart / resumable upload**, **pre-signed URL**

---

## Personal Notes (recurring gaps)
- [x] Vocabulary improving (used Redis, least-connections, stateless correctly).
- [ ] Still: precise names ("chunks" not "packets"; "JWT" not "IndexedDB").
- [ ] SIGNATURE LESSON (3 days running): **state must live in a shared store, NEVER on the server.**
