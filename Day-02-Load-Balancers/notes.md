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
  ends *at the LB*.) Precision: what is L7-only is **reading the HTTP inside** after decrypting. Some L4
  LBs can also decrypt TLS (e.g. AWS NLB has TLS listeners) but they still route only on IP/port. The
  three TLS modes — terminate / passthrough / re-encrypt — are in the pass-2 section below.
- **Caching** — keep a copy of common responses so it can answer without bothering a server.
- **Compression** — shrink responses before sending them out.
  - ⚠️ **Careful — these last two are NOT features of a *pure* load balancer.** A cloud LB like **AWS
    ALB** does **not** cache or compress; it only routes. Caching and compression are **reverse-proxy /
    CDN** features. You get them only when your L7 LB *is also* a reverse proxy — e.g. **Nginx** or
    **HAProxy** — which wear both hats. So the honest framing: "L7 LBs that double as a reverse proxy
    (Nginx) can cache/compress; a bare cloud LB (ALB) cannot."

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
  - ⚠️ **The trap with plain `hash(IP) % N`:** the `% N` is tied to the server *count*. The moment you add
    or remove a server, `N` changes, and **almost every user gets remapped to a different server** — so
    nearly everyone loses their sticky server (and their session with it) on any scale event. This exact
    pain is what **consistent hashing** (see the audit section below) was invented to fix.

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

## ⭐ Interview-completeness additions (audit pass)

These are the pieces an interviewer expects you to reach for once the basics above are solid. Same idea
each time: name the mechanism, know why it exists.

**1. LB high-availability: VIP + VRRP + keepalived (HIGHEST — "who balances the balancer?")**
We said "run multiple LBs with failover" — but *how* does a client's traffic actually jump to the standby
LB? The trick: clients never target a physical LB's IP. They target a **floating / virtual IP (VIP)** —
an address that isn't nailed to one machine. **VRRP** (Virtual Router Redundancy Protocol), run by a
daemon called **keepalived**, lets a standby LB **claim the VIP the instant the active LB dies**. It
announces the takeover with a **gratuitous ARP** (a broadcast that says "the VIP now lives at my MAC
address"), so the network reroutes packets to the standby within seconds. That gratuitous-ARP hand-off
*is* the mechanism behind **active-passive failover** — memorize the phrase "VIP + VRRP + keepalived."

**2. Connection draining / deregistration delay (zero-downtime deploys)**
When you remove a server — a deploy, a scale-down — you can't just kill it, or every request it's mid-way
through serving dies with it. **Connection draining** (AWS calls it **deregistration delay**) means: the
LB **stops sending new requests** to that server immediately, but **lets the in-flight ones finish** (up
to a timeout, e.g. 30–300s) before the server is actually shut down. Without this, every single deploy
drops live connections and users see errors. With it, deploys are invisible.

**3. Consistent hashing (in the LB context)**
The fix for the `hash(IP) % N` trap above. Instead of `% N`, you place **both the servers and the keys
(users) on a ring** (imagine a clock face of hash values); a key is served by the **next server clockwise**
on the ring. Now adding or removing a server only re-homes the keys **between it and its neighbor** —
roughly **1/N of keys move, not everyone**. **Virtual nodes** (each physical server placed at many points
on the ring) smooth out the distribution so no one server gets a fat arc. Interviewers ask for this **by
name** — it's the standard answer for "how do you keep affinity stable as the fleet scales?"

**4. Cloud product mapping + hardware vs software**
Know the concrete names, because interviewers switch to them fast:
- **Software LBs:** **Nginx**, **HAProxy**, **Envoy** — run on commodity servers, flexible, cheap.
- **Hardware LB:** **F5 Big-IP** — a dedicated physical appliance, very fast, very expensive, legacy/enterprise.
- **Cloud:** **AWS ALB = Application (L7)** — path / host / header routing. **AWS NLB = Network (L4)** —
  ultra-fast, routes on IP/port only. **Memorize: "ALB = L7, NLB = L4."**

**5. Health-check specifics**
"The LB pings each server" has more depth. **Active** health checks = the LB **probes** an endpoint (e.g.
`GET /health`) on a fixed **interval**. **Passive** health checks = the LB **watches real traffic** and
marks a server bad after **N consecutive errors**. To stop a flaky server from flip-flopping in and out
(**flapping**), you set **healthy / unhealthy thresholds** — e.g. "3 good checks to re-add, 2 bad to
remove." And distinguish **liveness** (is the process even up?) from **readiness** (is it *warmed up* —
DB connected, caches primed — and ready to take traffic?). A server can be live but not ready.

**6. GSLB / DNS-based global load balancing**
Everything above balances within one region. To route users **across regions** (e.g. Europe → the EU
data center, US → the US one), you use **GSLB (Global Server Load Balancing)**, usually done at the **DNS**
layer: the DNS server **returns different IPs** depending on the user's **geo, latency, or health**
(**AWS Route 53** routing policies do exactly this). ⚠️ **Limitation:** DNS answers are **cached for the
TTL** (time-to-live), so when a region dies, clients keep hitting the dead IP until their cache expires —
**failover is slow**. That's precisely why **Anycast** (one IP announced from many locations, the network
picks the nearest live one) is preferred when you need *fast* global failover.

**7. (polish) Two-tier L4→L7 and Power-of-Two-Choices**
- **Two-tier L4 → L7:** at massive scale you don't have *one* smart L7 LB — you put a **fast, dumb L4 LB
  in front** that just spreads connections across a **fleet of L7 LBs** behind it. The L4 tier handles raw
  volume; the L7 tier handles the smart routing. Best of both.
- **Power-of-Two-Choices:** true least-connections needs the LB to track live load on *every* server —
  expensive at scale. The cheap trick: **pick 2 servers at random, send to the less-loaded of the two.**
  Surprisingly, this lands **nearly as well as full least-connections** for a tiny fraction of the
  bookkeeping. A great answer to "how do you balance load cheaply across thousands of servers?"

---

## ⭐ Completeness additions (pass 2)

### 1. Where does HTTPS end? TLS termination vs passthrough vs re-encryption
HTTPS traffic arrives encrypted. The question is **who decrypts it** — and there are exactly three answers.
Picture a sealed letter arriving at a company mailroom (the LB):
- **TLS termination (offloading)** — the **LB decrypts** it, reads it, and forwards **plain HTTP** to the
  backend over the private network. The mailroom opens the letter, reads who it's for, and walks the open
  letter to the desk. ✅ The LB can do L7 routing (path/header/cookie), and backends are spared the crypto CPU;
  certificates live in one place. ❌ Traffic is unencrypted inside your network for the last hop.
- **TLS passthrough** — the LB does **not** decrypt; it forwards the encrypted bytes untouched and the
  **backend** does the TLS handshake. The mailroom passes the sealed envelope straight on. ✅ End-to-end
  encryption — the LB never sees plaintext (useful for strict compliance, or when the backend must see the
  client's certificate itself). ❌ The LB is blind to the content, so it can only route at **L4** — no path
  routing, no cookie stickiness (it can still route by the hostname in the unencrypted **SNI** field of the
  handshake on many proxies).
- **Re-encryption (TLS bridging / end-to-end TLS)** — the LB **decrypts, inspects/routes, then opens a new
  TLS connection** to the backend. The mailroom opens, reads, then re-seals in a fresh envelope for the internal
  trip. ✅ L7 smarts **and** encryption on every hop (common in "zero-trust" networks). ❌ Double the crypto
  work, and certificates to manage on the backends too.

Interview line: *"Default: terminate at the L7 LB. If compliance demands encryption in transit everywhere,
re-encrypt to the backends; use passthrough only when the LB must never see plaintext — accepting L4-only
routing."* (Service-to-service **mTLS** is Day 13; the security side is Day 23.)

### 2. WebSockets and other long-lived connections through an LB
Normal HTTP is "ask, answer, done" — a request occupies a server for milliseconds. A **WebSocket** (a chat
app, live game, stock ticker — protocol details on Day 12) is one connection that **stays open for minutes or
hours**, carrying messages both ways. That changes the LB's life in four ways:
- **It's sticky by nature.** The LB picks a backend **once, when the connection opens**, and every message
  then flows over that same connection to that same server. No cookie needed — but it also means the LB can't
  rebalance that user later.
- **Load gets uneven.** Round robin balances *new connections*, not *current load*. Add a fresh server and it
  gets zero existing connections — the old servers stay loaded until users happen to reconnect. So use
  **least connections** (count of open sockets ≈ real load) rather than round robin.
- **Draining takes ages.** Connection draining waits for in-flight work to finish — but a WebSocket may
  *never* finish on its own. Deploys therefore either wait a long drain timeout (AWS lets the deregistration
  delay go up to an hour) or the server **tells clients to reconnect** and closes them in batches.
- **Reconnect storms + idle timeouts.** If a server holding 50,000 sockets dies, 50,000 clients reconnect at
  once — clients need **exponential backoff with jitter** (Day 21). And LBs close connections that are silent
  for too long (AWS ALB's default idle timeout is 60 s), so apps send periodic **heartbeat/ping** messages.
- **Large uploads** are the same story in miniature: a long transfer pins one connection — which is why the
  notes above push big uploads **directly to S3 with presigned URLs**, bypassing the LB entirely.

Also, the LB must actually *support* the protocol: the WebSocket handshake is an HTTP request with an
`Upgrade: websocket` header, which L7 LBs like ALB, Nginx and HAProxy handle; an L4 LB just sees a long TCP
connection and forwards it.

### 3. The remaining algorithms, and how to choose
Adding to round robin, weighted RR, least connections, least response time, IP hash, P2C:
- **Random** — pick any server at random. With many requests it evens out statistically, needs zero state, and
  is the building block of P2C (pick two at random, keep the better).
- **Weighted least connections** — least connections, but divide each server's open-connection count by its
  weight, so a 2× machine is allowed 2× the connections before it counts as "busier."
- **Consistent hashing** (audit item 3) — as an *algorithm choice*: use it when the **same key should keep
  hitting the same server** because that server holds something useful for it — a cache shard, an in-memory
  game room, a user's local data — and you want that affinity to survive servers being added/removed.

Choosing (summary after the teaching above):
| Situation | Pick |
|---|---|
| Identical servers, short similar requests | Round robin (or random) |
| Mixed server sizes | Weighted RR / weighted least connections |
| Requests vary a lot in duration, long-lived connections | Least connections |
| Latency varies by server and you can measure it | Least response time |
| Thousands of servers, want least-conn quality cheaply | Power of two choices |
| Same key must reach the same server, fleet changes size | Consistent hashing |
| Quick stickiness with no cookies | IP hash (with the caveats below) |

### 4. The three ways to make a session sticky
- **LB-generated cookie** — the LB sets its own cookie on the first response ("you belong to server 3") and
  reads it on later requests. Works for any app, no code changes. (AWS ALB calls this duration-based stickiness.)
- **Application cookie** — your app sets its own session cookie and the LB is configured to stick on *that*
  cookie, so stickiness lasts exactly as long as the app's session.
- **IP hash** — no cookie at all; works for non-HTTP traffic too. But: many users behind one corporate/mobile
  **NAT share one IP** and all land on one server (uneven), and a phone switching from Wi-Fi to 4G **changes
  IP** and loses its server.
Whichever you use, the downsides from the Sticky Sessions section still apply — the real fix stays
**stateless servers + shared session store**.

### 5. Global LB: DNS-based vs Anycast-based, and the limits of DNS load balancing
Audit item 6 introduced GSLB. The two ways to do it:
- **DNS-based GSLB** — the authoritative DNS returns a different IP per user (geo/latency/weighted/health —
  the Route 53 policies on Day 4). Simple and flexible. Its **limits**: (1) answers are **cached for the TTL**
  (and some resolvers ignore it), so failover is slow; (2) DNS sees the **resolver's** location, not the user's,
  so a user on a far-away public resolver may get a poor choice; (3) the client **doesn't know** whether the IP
  it cached is healthy; (4) big resolvers serve thousands of users the same answer, so traffic splits are lumpy.
- **Anycast-based** — one IP announced from every region via BGP; the network delivers each packet to the
  nearest site, and if a site withdraws its announcement, traffic shifts **without waiting on any DNS cache**.
  Faster failover, but you need to run your own network edge (or buy it — e.g. AWS Global Accelerator,
  Cloudflare).

### 6. The full request path — DNS → GSLB → L4 → L7 → backend
Tie it all together the way an interviewer wants to hear it:
1. **DNS / GSLB** resolves `api.example.com` to the **best region's** address (or an Anycast IP).
2. A **L4 LB** (e.g. NLB) in that region takes the raw TCP connections — huge volume, very fast — and spreads
   them across a fleet of…
3. **L7 LBs / reverse proxies** (ALB, Nginx, Envoy) that **terminate TLS**, read the path/headers, and route
   `/api/*` to the API service, `/video/*` to the video service…
4. …which pick a **healthy backend instance** using an algorithm (least connections, P2C…).
5. The backend is **stateless** and reads session/data from Redis and the DB.
At every layer: **health checks** remove dead members, and **every layer runs redundant** (no SPOF).

### 7. Cloud mapping, complete
AWS's **Elastic Load Balancing (ELB)** is the umbrella name for a family: **ALB** (L7 HTTP/HTTPS/WebSocket),
**NLB** (L4 TCP/UDP/TLS, very high throughput, static IPs), **GWLB** (Gateway LB — for inserting firewall /
inspection appliances), and the old **Classic LB** (legacy). Other clouds have equivalents (e.g. GCP Cloud Load
Balancing, Azure Load Balancer = L4 / Application Gateway = L7).

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
