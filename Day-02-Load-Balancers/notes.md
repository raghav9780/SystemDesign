# Day 2 — Load Balancers
**Phase 1: Foundations** | Score: 8/10

---

## What a Load Balancer Does
- Sits between users and the server fleet; every request hits it first.
- 3 jobs: (1) **distribute traffic**, (2) **health checks** (drop dead servers), (3) **hide the fleet** (users see one address).
- Removes the Day 1 web-tier SPOF.

---

## L4 vs L7

### L4 (Transport layer)
- Routes by **IP + TCP/UDP port** only. Doesn't read request content.
- ✅ Very fast, low overhead.  ❌ Dumb — no content-based decisions.

### L7 (Application layer) — most modern web apps
- Routes by **content**: URL path, headers, cookies.
- Can do path-based routing (`/video/*` → video servers, `/api/*` → API servers),
  SSL termination, caching, compression.
- ✅ Smart.  ❌ More CPU → slightly slower. (Nginx, AWS ALB)

---

## Routing Algorithms
- **Round Robin** — 1,2,3,1,2,3. Assumes all servers/requests equal.
- **Weighted Round Robin** — beefier servers get more (weights).
- **Least Connections** — fewest active connections wins. ⭐ Best when requests vary
  in duration (e.g. heavy long-lived video vs light API).
- **Least Response Time** — fastest responder wins.
- **IP Hash** — hash IP → same server every time (a way to "stick" users).

---

## Sticky Sessions (Session Affinity) 🍯
- LB pins a user to the SAME server for their whole session (cookie / IP hash).
- ✅ Easy fix for stateful apps.
- ❌ Breaks load balancing benefits: if that server dies, its users lose sessions;
  traffic gets uneven.
- ✅✅ BETTER FIX: make servers **stateless** → store sessions in shared store (Redis).
  Any server serves any user; sticky sessions become unnecessary.

---

## SPOF Ladder (KEY LESSON)
- Every component you add to fix a SPOF can BECOME a SPOF. Ask each time.
- Web servers → **LB is a SPOF** → run multiple LBs (active-active/passive + failover).
- Redis session store → **Redis is a SPOF** → Redis replication + cluster + failover.
- Fix for SPOF = **add redundancy**, NOT delete the component.

---

## Large File Uploads (2GB video, server may be killed mid-transfer)
- **Chunking / multipart upload** — split file into app-level CHUNKS (5–10MB),
  upload each independently. (NB: "chunks" = app-level; "packets" = TCP/OS-level.)
- **Resumable upload** — track last acknowledged chunk; resume at N+1 on failure.
- CRITICAL: chunks must go to shared **object storage (S3)**, NOT the server's local disk.
  S3 tracks received chunks centrally → any server can resume any upload → stateless web tier.
- **Pro move:** client gets a **pre-signed URL** and uploads chunks straight to S3,
  bypassing web servers for the heavy transfer. Servers just give permission + record metadata.

---

## Vocabulary
- L4 / L7, SSL termination, path-based routing
- Session affinity / sticky sessions
- Least Connections, IP Hash
- Multipart / resumable upload, pre-signed URL

---

## Personal Notes (recurring gaps)
- [x] Vocabulary improving (used Redis, least-connections, stateless correctly).
- [ ] Still: precise names ("chunks" not "packets"; "JWT" not "IndexedDB").
- [ ] SIGNATURE LESSON (3 days running): **state must live in a shared store, NEVER on the server.**
