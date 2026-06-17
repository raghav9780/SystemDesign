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

---

## B. Anycast — one IP, many locations 🌍

Normally one IP belongs to one machine. **Anycast** breaks that rule on purpose: the *same* IP is
**announced from many data centers at once**, and the internet's routing automatically sends you to the
**nearest/healthiest** one. It's like a hotline number that rings through to whichever call center is
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

## Personal Notes (recurring gaps)
- [ ] Prescribe the SPECIFIC ACTION (Q3: "lower the TTL in advance", not just "TTL caches it").
- [ ] FINISH THE FULL LOOP (Q4: include LB→cache→DB→render, not just up to HTTP request).
- [x] Naming discipline now consistent across the whole answer ✅.
