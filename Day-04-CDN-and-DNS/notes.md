# Day 4 — CDN & DNS
**Phase 1: Foundations** | Score: 8/10

> **One-sentence idea:** DNS turns a human name (`google.com`) into a machine address (an IP), and a CDN
> keeps copies of your files **near** the user — so the whole day is really about two kinds of "look it up
> close to you instead of trekking to the source": **DNS caches the address**, the **CDN caches the content**.
> Think of DNS as the **phone book** (name → number) and the CDN as **local warehouses** of a big store
> (the same product stocked in your city so you don't wait for it to ship from headquarters).

---

## A. DNS — the internet's phone book

Computers don't talk to names, they talk to **IP addresses** (e.g. `142.250.x.x`). You type `google.com`
because humans can't remember numbers; **DNS (Domain Name System)** is the lookup service that translates
that name into the IP your browser actually connects to. Same job as looking up a contact's name to get
their phone number.

### The lookup chain (what happens on a cache miss)

The clever part is that nobody asks the "real" source every time — there's a chain of caches checked first,
and only if they all miss does the request walk all the way up to the authoritative source. Picture asking
"do you have this person's number?" to closer and closer-but-bigger phone books until someone does:

1. **Browser cache** — did *this browser* look it up recently?
2. **OS cache** — did *this computer* (any app) look it up recently?
3. **Router cache** — did anyone on *this home/office network* look it up recently?
4. **ISP resolver** — your internet provider's DNS server; the first "real" server, and it caches for many customers.
5. **Root server** — the top of the tree. It doesn't know `google.com`, but it knows who runs `.com`.
6. **TLD server** (Top-Level Domain, e.g. `.com`) — knows which nameserver is authoritative for `google.com`.
7. **Authoritative nameserver** — the source of truth for that domain; it **returns the IP**.
8. The answer is then **cached at each level on the way back**, for as long as the record's **TTL** allows.

⭐ Key intuition: the first four steps are all caches. The expensive root → TLD → authoritative walk only
happens when every cache misses. That's *why* the internet isn't crushed under DNS traffic.

### TTL (Time To Live) — how long a cached answer is trusted

**TTL** is a timer attached to each DNS record saying "you may cache me for this many seconds before you
must re-fetch." It's a direct trade-off:

- **Low TTL (e.g. 60s):** changes spread fast (caches expire and re-ask quickly), but you pay for *many more*
  lookups.
- **High TTL (e.g. 24h):** very few lookups (cheap, fast for users), but a change takes up to a day to
  reach everyone because old answers stay cached.

⭐ TTL is **THE lever** for failover and migrations — because it controls how quickly the world will pick up
a new IP. Remember this; it's the whole trick behind the migration section below.

### Record types (the kinds of entries in the phone book)

A DNS record isn't always "name → IP." The common types:

- **A** — maps a name to an **IPv4** address (the classic `142.250.x.x`).
- **AAAA** — maps a name to an **IPv6** address (the newer, longer address format).
- **CNAME** — an **alias**: points one name at *another name* (e.g. `www.site.com` → `site.com`) instead of an IP.
- **MX** — **Mail eXchange**: says which server receives email for the domain.
- **NS** — **NameServer**: says which server is authoritative for the domain.
- (The rest of the set — **SOA, TXT, PTR, SRV, CAA** — plus the **CNAME-at-apex** problem and its
  **ALIAS/ANAME** fix are in the pass-2 section at the end.)

---

## B. Anycast — one IP, many locations 🌍

Normally one IP belongs to one machine. **Anycast** breaks that rule on purpose: the *same* IP is
**announced from many data centers at once**, and the internet's routing automatically sends you to the
**nearest** one — "nearest" in network terms (the shortest BGP route), which is usually but not always the
geographically closest. "Healthiest" happens because a failing site **stops announcing** the IP, so routers
stop sending traffic there. It's like a hotline number that rings through to whichever call center is
closest to you — you dial one number, but who answers depends on where you are.

Why it matters — three wins fall out of it for free:

- ✅ **Low latency:** you're routed to the geographically nearest copy.
- ✅ **Built-in failover:** if one data center dies, routing simply steers you to the next-nearest one
  announcing that same IP — no DNS change needed.
- ✅ **DDoS absorption:** a flood of attack traffic gets spread across many locations instead of drowning one.

This is how public DNS resolvers like **8.8.8.8** and CDNs serve the whole planet from a single address.

---

## C. CDN — Content Delivery Network 📦

A **CDN** is a network of **edge servers** spread around the world that **cache your content close to users**.
Instead of every user fetching a file from your one **origin** server, they grab it from a nearby edge.

The reason this matters is pure physics — distance is latency. A round trip from Virginia to Mumbai is
~200ms; a CDN edge already in Mumbai answers in ~10ms. Same file, 20× faster, just because it's near.

⭐ What belongs on a CDN: **static** content — images, video, CSS, JS, fonts — files that are the *same for
everyone*. You do **not** put dynamic or user-specific content there (your personalized feed, your account
balance), because that has to be freshly computed per request and isn't shareable across users.
(Nuance: dynamic traffic can still *pass through* a CDN uncached and get faster — see "dynamic acceleration"
in the pass-2 section.)

### How the edge gets the content — fill methods

- **Pull CDN** — the edge starts empty. On the **first request (a miss)** it fetches the file from your
  origin, caches it, and serves all later requests from cache. It's *lazy* — content is pulled only when
  someone asks (exactly the **cache-aside** pattern from Day 3). This is the **most common** setup.
- **Push CDN** — **you upload** the content *directly to the CDN* ahead of time, and the CDN distributes it
  to its edges. Best for **large files you already know will be hot** — e.g. a video release everyone will
  hit at once, where you don't want the first viewers to suffer the slow origin fetch. (NB: you push
  *straight to the CDN* — there's no separate "master server" in between.)

Intuition: **pull** = stock the shelf only when a customer asks for the item; **push** = pre-stock the shelf
because you know there's a rush coming.

### CDN invalidation — when the content changed but stale copies linger

The flip side of caching: once an edge has a file, it'll keep serving the **old** version until told
otherwise. Two ways to deal with it:

- **TTL expiry** — wait for the cached copy's timer to run out, then the edge re-fetches the fresh one.
- **Versioned URLs / cache-busting** — change the *URL itself* when content changes (`style.v2.css`,
  `image.jpg?v=123`). Since it's a *new* URL, the edge has never seen it → it's a guaranteed miss → it
  fetches the new file immediately. No waiting for any timer.

⭐ For something like a **profile photo**, versioned URLs give an **instant update with zero propagation
wait** — point the page at the new URL and every user sees the new image right away, because the old
cached URL is simply abandoned.

### Players
Cloudflare, Akamai, AWS CloudFront, Fastly.

---

## D. Zero-downtime DNS migration (moving to a NEW IP) ⭐

Say you're moving your service to a new server with a new IP, and you can't afford downtime. The danger is
that resolvers everywhere have your **old** IP cached (for up to its TTL), so the moment you flip, some
users still get sent to the old box. The fix is to use the TTL lever *in advance* and overlap the two
servers. The sequence:

1. **Lower the TTL to ~60s DAYS IN ADVANCE.** This is the non-obvious step: you must do it *before* the
   migration so the old high-TTL records (which were cached long) all expire and get replaced by short-TTL
   ones. Now the world will pick up a change within ~60s instead of hours.
2. **Flip DNS to the new IP.**
3. **Run BOTH old and new servers in parallel** during the cutover — so whichever IP a resolver still has,
   the user is served correctly either way.
4. After the new IP has propagated everywhere, **raise the TTL back up** (back to cheap, fewer lookups).

The catch: **stubborn ISPs may ignore your TTL** and hold the old IP longer than they should → so you keep
the old server alive (or redirect from it) for a grace period rather than killing it on schedule.

### ⭐ Stubborn-ISP follow-up (ISP ignores TTL, caches old IP for ~6h) — RESOLVED

The scenario: you did everything right, but a misbehaving ISP keeps handing the **old IP** to its customers
for hours past your TTL. What now?

- ❌ **DON'T just delete the old server.** Users whose resolver still hands them the stale IP would hit a
  dead address → **connection refused** → the app is simply *down* for them.
- ❌ **MYTH to kill:** a failed TCP connection does **NOT** make a resolver refresh its DNS cache. People
  assume "if the old IP fails, the client will just re-look-up and get the new one" — it won't. DNS caching
  and TCP failure are unrelated layers; the resolver keeps serving the stale IP until its (ignored) timer
  finally expires.
- ✅ **Keep the old server alive as a proxy / 301-redirect** to the new backend. Then even users still
  landing on the stale IP get **forwarded to the right place** → correct service, zero broken experience.
- ✅ **Cleanup = "DRAIN, don't YANK."** Don't decommission the old IP on a fixed date — **monitor the
  traffic** still hitting it and shut it down only when that traffic falls to ~0.
- **PRINCIPLE:** ⭐ **you don't own the client's DNS cache**, so the old endpoint must keep working until
  *nobody* is using it anymore — not until *you've decided* it's over. (This exact idea reappears in
  **blue-green deployments** and **API versioning**: stand up the new thing, keep the old one serving until
  usage drains, then retire it.)

---

## E. 🎬 "What happens when you type a URL" — THE screening answer

This is the classic interview opener, and it ties the whole day (plus earlier days) together. The rule is to
**finish the entire loop** — all the way to the page rendering — not stop at "the HTTP request is sent."

1. **DNS resolve** — browser cache → OS → resolver → root → TLD → authoritative nameserver returns the IP
   (and **Anycast** quietly routes you to the nearest edge/resolver).
2. **TCP handshake** — establish the connection: **SYN → SYN-ACK → ACK** (the three-way "hello, ready? ready.").
3. **TLS handshake** — for HTTPS, the encryption key exchange so the connection is private.
4. **HTTP GET** — the actual request for the page is sent.
5. **Load balancer (Day 2)** routes the request to a server → the server checks its **cache (Day 3)**, hitting
   the **DB only on a miss** → and the **CDN (Day 4)** serves the static assets.
6. **Browser renders** the returned HTML, then fetches CSS/JS/images (many of which come from the **CDN**).

⭐ FINISH THE WHOLE LOOP — include the response and the render, not just the request.

---

## ⭐ Interview-completeness additions (audit pass)

**1. Recursive vs iterative resolution (name the mechanism).** When you type a name, you make exactly *one*
query — to your **recursive resolver** (your ISP's, or a public one like `8.8.8.8`). It promises to come back
with a *final* answer and does all the legwork on your behalf. How does it get that answer? By making a series
of **iterative** queries: it asks a **root** server ("who handles `.com`?"), then the **TLD** server ("who's
authoritative for `google.com`?"), then the **authoritative nameserver** ("what's the actual IP?"). So name the
two halves in an interview: **client ↔ resolver is recursive** ("you go find it for me"), **resolver ↔
root/TLD/authoritative is iterative** ("here's the next server to ask"). Same chain from section A, now with the
mechanism labelled.

**2. Negative caching (NXDOMAIN).** Resolvers don't only cache *successes* — they cache **failures** too. When a
name genuinely doesn't exist, the authoritative server returns **NXDOMAIN** ("no such domain"), and the resolver
caches that *negative* answer (per RFC 2308 its lifetime is the **smaller of the SOA record's own TTL and the
SOA's "minimum" field** — the SOA record is explained in the pass-2 section). So
if a bot or a fat-fingered typo hammers a nonexistent name, the resolver answers "nope" straight from cache
instead of walking the whole root → TLD → authoritative chain every time. This is the DNS analogue of the
**cache-penetration** problem from Day 3 (caching the "not found" so misses don't stampede the source).

**3. HTTP Cache-Control headers (the Day 3 ↔ Day 4 bridge).** How does a browser or CDN actually *know* how long
to keep a cached file? The origin tells them via the **`Cache-Control`** response header — this is the missing
link between "caching" (Day 3) and "CDN/edge caching" (Day 4). Key directives: **`max-age`** = how long the
*browser* may cache; **`s-maxage`** = how long a *shared* cache (the CDN) may cache — it overrides `max-age` at
the edge; **`public`** = any cache may store it, **`private`** = only the browser may (never a shared CDN — think
user-specific data); **`no-cache`** = you may store it but must **revalidate** before reuse; **`no-store`** =
never write it down at all. Revalidation is made cheap by **`ETag`** (a content fingerprint the origin sends):
the client sends it back as **`If-None-Match`**, and if unchanged the origin replies **`304 Not Modified`** with
an empty body — you skip re-downloading the whole file. And **`stale-while-revalidate`** lets the edge serve the
*stale* copy instantly while it refreshes in the background — no user waits on the fetch.

**4. GeoDNS / routing policies + GSLB.** Section A treated DNS as returning *the* IP — but an authoritative server
can return **different IPs to different resolvers on purpose**. Common policies: **geo-based** (European resolvers
get the Frankfurt IP, US resolvers get Virginia), **latency-based** (return whichever data center is fastest for
that resolver), **weighted** (send 5% of answers to a new version — a **canary** split), plus **health-checked
failover** (stop handing out an IP once its endpoint fails a health check). Together this is **DNS-based Global
Server Load Balancing (GSLB)** — load balancing at the *name-resolution* layer, across regions. The caveat ties
straight back to **TTL**: answers are cached for the TTL, so GeoDNS is **coarse** and can **misroute** a user
whose resolver is far from them or holding a stale answer. That imprecision is exactly *why* CDNs lean on
**Anycast** (section B) for fine-grained, per-request routing instead.

**5. Origin shield / multi-tier caching + hit ratio.** A pull CDN (section C) has a hidden failure mode: when a
file is *cold*, **every edge PoP** that gets a first request fetches it from your origin independently — so a
brand-new object can trigger **N origin fetches**, one per PoP, all at once. The fix is an **origin shield**: a
designated **mid-tier cache** that all edges consult *before* going to the origin. Now a cold object causes just
**one** origin fetch (the shield's), and every edge fills itself *from the shield*. This **protects the origin**
from stampedes and **raises the overall cache hit ratio**, since the shield aggregates misses that would
otherwise each hit origin.

**6. CDN security — signed URLs / signed cookies.** How do you serve **private or paywalled** content (a paid
video, a customer's invoice) through a *public* CDN edge that anyone can reach? **Signed URLs**: the origin
issues a URL carrying a **time-limited, cryptographically signed token**; the edge **validates the signature and
expiry** before serving, so the link can't be **shared or replayed** once it expires. (**Signed cookies** do the
same for a whole set of files rather than one URL.) CDNs also **terminate TLS at the edge** (the HTTPS handshake
from section E happens at the nearby PoP, not your distant origin) and sit in front as a security layer providing
**WAF, DDoS mitigation, and bot protection**.

**7. Active purge / tag (surrogate-key) invalidation.** Section C gave two invalidation options — wait out the
**TTL** or use **versioned URLs**. There's a third: **active purge**, where you *tell* the CDN to drop cached
copies *now*. You can purge a **single URL**, or — more powerfully — attach **tags (surrogate keys)** to objects
(e.g. tag every page and asset for `product-123`) and then **invalidate every edge copy carrying that tag in one
API call**. Trade-off vs versioned URLs: **versioned URLs** update **instantly with no purge needed** (the old
URL is just abandoned) but force you to **change every reference** to the file; **tag purge** needs an explicit
call and brief propagation but requires **no URL changes** — great for content you can't easily re-reference
everywhere.

---

## ⭐ Completeness additions (pass 2)

### 1. The DNS chain, with every piece named
Section A's 8 steps, with the official names an interviewer may use:
- **Stub resolver** — the tiny DNS client built into your OS. It doesn't hunt for answers itself; it just
  asks one configured server and waits. Before asking, the OS also checks the **hosts file**
  (`/etc/hosts` on Linux/Mac) — a local hand-written name→IP list that overrides DNS.
- **Router cache** — many home/office routers run a small DNS forwarder that caches answers; not all do.
- **Recursive resolver** — your ISP's resolver or a public one (`8.8.8.8` Google, `1.1.1.1` Cloudflare). It
  does the root → TLD → authoritative legwork and caches the results for everyone it serves.
- **Root → TLD → authoritative nameserver** — the iterative walk (audit item 1).
- Every layer caches **positive** answers (here's the IP) and **negative** answers (NXDOMAIN — audit item 2),
  each for its TTL.

### 2. The full record-type set — and why a CNAME can't live at the apex
Section A had A, AAAA, CNAME, MX, NS. The rest of the set, each with what it's for:
- **SOA (Start of Authority)** — the zone's "header card." Exactly one per zone, at the top. It names the
  primary nameserver and an admin email, carries a **serial number** (bumped on every change so secondary
  nameservers know to re-copy the zone), refresh/retry/expire timers for those secondaries, and the
  **minimum** field used for negative-caching TTL.
- **TXT** — free-form text attached to a name. Sounds useless, but it's how you **prove you own a domain**
  ("add this TXT value and we'll verify it" — Google, AWS certificates, etc.) and how email anti-spoofing works:
  **SPF** (which servers may send mail for me), **DKIM** (public key to verify mail signatures), **DMARC**
  (policy if those checks fail).
- **PTR (Pointer)** — **reverse DNS**: IP → name, the opposite of an A record. Lives in a special zone
  (`in-addr.arpa` for IPv4). Main real-world use: mail servers check that a sending IP has a sensible PTR
  before trusting it.
- **SRV (Service)** — says **where a service lives, including the port**: `_sip._tcp.example.com` → priority,
  weight, port 5060, host `sip1.example.com`. Lets clients discover "which host *and port* runs service X"
  (used by SIP, XMPP, LDAP/Active Directory, some service-discovery systems).
- **CAA (Certification Authority Authorization)** — says **which certificate authorities may issue TLS
  certificates** for your domain (e.g. only Let's Encrypt). CAs must check it before issuing, so a random CA
  can't be tricked into minting a cert for you.

**The CNAME-at-apex problem.** The **apex** (or "zone root," "naked domain") is `example.com` itself, as opposed
to `www.example.com`. Suppose your site is hosted on a load balancer or CDN whose address is a *name* (e.g.
`my-lb-123.elb.amazonaws.com`) whose IPs change over time — you'd naturally want `example.com CNAME
my-lb-123...`. **You can't.** The DNS rule (RFC 1034) is: **if a name has a CNAME, it may have no other records
at all** — because a CNAME means "this name is *entirely* an alias for that other name; go ask there for
everything." But the apex **must** have an **SOA** and **NS** records (that's what makes it a zone), and usually
**MX** (email) and **TXT** too. A CNAME there would contradict them, so it's forbidden.

**The fix: ALIAS / ANAME / CNAME flattening.** DNS providers invented a record that *behaves* like a CNAME but
*answers* like an A record: the **authoritative server itself looks up the target's current IPs** and returns
plain **A/AAAA** records to the client. To the outside world the apex just has A records (legal next to
SOA/NS/MX), while you still get "follow that name" convenience. Names differ by vendor: **Route 53 "alias"
records**, **ALIAS/ANAME** at several DNS providers, **CNAME flattening** at Cloudflare. Interview line: *"Use a
CNAME for `www`; for the bare domain use an ALIAS/flattened record, because a CNAME can't coexist with the
apex's SOA and NS records."*

Summary (reference only, after the teaching above):
| Record | Maps | Typical use |
|---|---|---|
| A / AAAA | name → IPv4 / IPv6 | the normal lookup |
| CNAME | name → another name | `www` → CDN/LB hostname (not at apex) |
| ALIAS/ANAME | apex name → name, answered as A | `example.com` → LB/CDN |
| NS | zone → its nameservers | delegation |
| SOA | zone metadata | serial, timers, negative-cache TTL |
| MX | domain → mail servers | email delivery |
| TXT | name → text | domain verification, SPF/DKIM/DMARC |
| PTR | IP → name | reverse DNS, mail checks |
| SRV | service → host + port | service discovery |
| CAA | domain → allowed CAs | certificate-issuance control |

### 3. Route 53 routing policies — the complete list
Audit item 4 described geo/latency/weighted/failover in general. AWS Route 53's actual policy names (8 of them):
- **Simple** — one record, one answer (or several IPs returned together, no health checks). The default.
- **Weighted** — split answers by weights, e.g. 95/5 → **canary** releases or gradual migrations.
- **Latency** — answer with the AWS Region that has the **lowest measured latency** for that user's network.
- **Geolocation** — answer by **where the user is** (continent/country/state) — for data-residency rules,
  language-specific content, licensing.
- **Geoproximity** — answer by **distance between user and your resources**, with a **bias** knob to grow or
  shrink a region's catchment area (shift traffic toward a bigger region).
- **Failover** — **active-passive**: return the primary while its **health check** passes, the secondary when it
  fails.
- **Multivalue answer** — return **up to 8 healthy records** at random, so the client has several to try —
  "DNS round robin with health checks."
- **IP-based** — route by the **user's IP range** (CIDR blocks you list), e.g. send a known ISP's users to a
  specific endpoint.
Health checks can be attached to most of these, so an unhealthy endpoint is dropped from answers — that's the
**GSLB** behaviour. One more detail: to guess the user's location, Route 53 can use the **EDNS Client Subnet**
extension (EDNS0), in which the resolver forwards part of the *user's* IP — this softens the "resolver is far
from the user" problem from audit item 4.

### 4. Anycast vs GeoDNS — the comparison interviewers ask for
Both send users somewhere close, at different layers:
- **GeoDNS** decides **once, at lookup time**, by handing different IPs to different resolvers. Flexible
  (any rule you like: weights, geography, health) but **coarse** — it sees the resolver not the user, and the
  answer is stuck in caches for the TTL, so failover waits on TTLs.
- **Anycast** decides **per packet, in the network**: one IP everywhere, BGP routing delivers to the nearest
  site. **Fast failover** (a dead site withdraws its route — no cache to wait out) and natural **DDoS
  spreading**, but less control (you can't say "send exactly 5% to Frankfurt") and it needs your own network
  presence. It's best for short, stateless exchanges like DNS queries and CDN edges; a route change mid-way can
  move a long TCP connection to a different site, which CDNs engineer around.
- Real systems combine them: e.g. Anycast to reach the nearest CDN edge, GeoDNS/latency routing to pick the
  origin region.

### 5. CDN fundamentals: the cache key, `Vary`, and dynamic acceleration
- **Cache key** — how the edge decides "do I already have *this*?" By default it's roughly **host + path +
  query string**. If your URLs carry junk query params (`?utm_source=...`), each variant becomes a separate cache
  entry and your **hit ratio** collapses — so CDNs let you **strip or ignore** params in the key. Conversely, if
  the response depends on something *not* in the key, you risk serving one user's variant to another.
- **`Vary` header** — the origin's way of saying "my response differs by this request header, so include it in
  the key." `Vary: Accept-Encoding` → keep separate gzip and brotli copies (safe and common). But `Vary:
  User-Agent` or `Vary: Cookie` multiplies variants into thousands → hit ratio near zero. Rule: vary on as few,
  low-variety headers as possible.
- **Static vs dynamic — dynamic acceleration.** You don't *cache* a personalised feed — but routing it through
  the CDN still helps: the user's TCP/TLS handshake happens with the **nearby edge** (short round trips), the
  edge keeps **warm, persistent connections** to your origin over the CDN's well-tuned backbone, and the WAF/DDoS
  protection still applies. So: static = cached at the edge; dynamic = *proxied* through the edge, not cached.

### 6. Edge computing — running code at the CDN, not just caching
A normal CDN can only hand back stored files. **Edge computing** lets you run **small pieces of your own code
in the edge PoPs**, right next to the user — think of the local warehouse not just storing products but having a
small workshop that can customise them on the spot.
- **What it's good for:** **stateless, latency-sensitive** logic that would otherwise cost a round trip to the
  origin — checking an auth token / signed URL, redirects and URL rewrites, A/B-test bucketing, adding security
  headers, geo-based personalisation ("show prices in ₹ for India"), bot filtering, resizing images.
- **The products:** **Cloudflare Workers**, **AWS Lambda@Edge** (and the lighter **CloudFront Functions**),
  **Fastly Compute**.
- **Why startup time is the whole story — isolates vs containers.** Edge code runs on demand in hundreds of
  locations, so how fast a fresh copy can start matters enormously.
  - **Cloudflare Workers** run in **V8 isolates** — the same sandboxing a browser uses to keep tabs apart. An
    isolate is created *inside an already-running* process, so it starts in **milliseconds or less** with tiny
    memory; Cloudflare says isolates start roughly **100× faster than a Node process in a container/VM**, which
    effectively removes the classic **cold start**.
  - **Fastly Compute** reaches a similar result with **WebAssembly** (not V8), and Fastly reports instance
    start-up in **microseconds**.
  - **Container/VM-based functions** (classic AWS Lambda, and Lambda@Edge) must boot a runtime for a new
    instance — a **cold start** of very roughly **~100 ms to 1 s+** depending on language and package size —
    then stay warm for later requests.
- **The constraints (why the origin still exists):** edge functions get **tight CPU-time and memory limits**,
  there's **no local durable state** (each request may hit a different PoP or a fresh isolate), and edge
  key-value stores are typically **eventually consistent** (e.g. Cloudflare Workers KV). So the split is:
  **edge = stateless, fast, per-request decisions; origin = stateful work** — databases, transactions,
  anything needing strong consistency.
Interview line: *"I'd validate the token and do the A/B bucketing at the edge to save a round trip, but keep
the checkout logic and DB writes at the origin."*

### 7. Pull vs push — the two classic scenario answers
- **News site** (many articles, constantly changing, only some go viral): **pull** — the edge fetches whatever
  readers actually request; the long tail never wastes edge storage; short TTLs + purge on edit.
- **Video platform launching a big release** (huge files, predictable demand): **push/pre-warm** the popular
  files to edges before launch so the first million viewers don't all miss at once — usually with an **origin
  shield** behind, and **pull** for the long tail of old videos. Real systems mix both.

### 8. Zero-downtime cutover — the "validate" step
Section D's sequence, with the step people forget: lower TTL (24–48 h ahead, i.e. at least one *old* TTL before)
→ switch DNS during the low-TTL window → **validate** (watch error rates and traffic shifting to the new IPs,
keep the old servers serving) → raise the TTL back up once you're confident → drain and retire the old endpoint.

### 9. (Senior) Multi-CDN and egress cost
Two things that come up once you're big:
- **Egress cost.** Clouds charge for data **leaving** them ("egress"), and for media-heavy apps bandwidth can be
  one of the biggest bills. Every CDN cache hit is a byte your origin *didn't* send — so **hit ratio is also a
  cost metric**, and origin shields + long TTLs + versioned URLs directly lower the egress bill.
- **Multi-CDN.** Using **two or more CDNs** at once, steered by DNS (weighted or performance-based routing).
  Why: **availability** — one CDN having a global incident takes you down (this has happened, e.g. Fastly's
  widespread outage in June 2021); **performance** — different CDNs are stronger in different countries; and
  **price leverage**. Costs: config, purge, and security rules must be kept identical across vendors, each CDN
  has its own cold cache (lower combined hit ratio), and debugging spans two vendors.

### 10. LLD bonus — "design a DNS cache"
A small low-level design that reuses the whole day:
- **Store:** a hash map `name+type → (records, expires_at)`; on lookup, if `now < expires_at` serve it, else
  treat as a miss and resolve upstream.
- **TTL eviction:** expire lazily on read, plus a periodic sweep (or a min-heap ordered by `expires_at`) so dead
  entries don't eat memory.
- **Size bound:** cap the entry count and evict **LRU** when full (Day 3).
- **Negative caching:** store NXDOMAIN answers too, with the SOA-derived negative TTL.
- **Stampede protection:** if 1,000 queries for the same uncached name arrive together, send **one** upstream
  query and fan the answer out (request coalescing, Day 3).

---

## Personal Notes (recurring gaps)
- [ ] Prescribe the SPECIFIC ACTION (Q3: "lower the TTL in advance", not just "TTL caches it").
- [ ] FINISH THE FULL LOOP (Q4: include LB→cache→DB→render, not just up to HTTP request).
- [x] Naming discipline now consistent across the whole answer ✅.
