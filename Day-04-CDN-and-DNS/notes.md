# Day 4 — CDN & DNS
**Phase 1: Foundations** | Score: 8/10

---

## DNS — the internet's phone book
- Translates name (google.com) → IP (142.250.x.x).
- Lookup chain (on cache miss):
  Browser cache → OS cache → Router cache → ISP resolver →
  Root server → TLD server (.com) → Authoritative nameserver → returns IP → cached by TTL.

### TTL (Time To Live)
- How long a DNS record may be cached before re-fetch.
- Low TTL (60s) = fast propagation, more lookups. High TTL (24h) = fewer lookups, slow to update.
- ⭐ THE lever for failover & migrations.

### Record types
- A (→IPv4), AAAA (→IPv6), CNAME (alias→name), MX (mail), NS (nameserver).

---

## Anycast — one IP, many locations 🌍
- Same IP announced from many data centers; routing sends you to nearest/healthiest.
- Powers DNS resolvers (8.8.8.8) and CDNs.
- ✅ Low latency, built-in failover, DDoS absorption.

---

## CDN — Content Delivery Network 📦
- Global edge servers caching content CLOSE to users.
- Virginia→Mumbai ~200ms; CDN edge in Mumbai ~10ms.
- Cache STATIC content: images, video, CSS, JS, fonts. (Not dynamic/user-specific.)

### Fill methods
- **Pull CDN** — edge fetches from origin on first miss, then caches (lazy, like cache-aside). Most common.
- **Push CDN** — YOU upload content directly to the CDN, which distributes to edges.
  Good for large files known to be hot (video releases). (NB: push straight to CDN, no "master server".)

### CDN invalidation (content changed, stale copies linger)
- TTL expiry, OR **versioned URLs / cache-busting** (`style.v2.css`, `image.jpg?v=123`).
- ⭐ For profile photos: versioned URLs = instant update, zero propagation wait.

### Players: Cloudflare, Akamai, AWS CloudFront, Fastly.

---

## Zero-downtime DNS migration (NEW IP) ⭐
1. **Lower the TTL to ~60s DAYS IN ADVANCE** (so resolvers don't hold old IP long).
2. Flip DNS to the new IP.
3. **Run BOTH old + new servers in parallel** during cutover (either IP works).
4. After propagation, raise TTL back up.
- Stubborn ISPs may ignore TTL → keep old server alive / redirect for a grace period.

### Stubborn-ISP follow-up (ISP ignores TTL, caches old IP 6h) ⭐
- ❌ DON'T just delete the old server → users on stale IP get connection-refused (app down for them).
- ❌ MYTH: a failed TCP connection does NOT make a resolver refresh its DNS cache.
- ✅ Keep old server alive as a **proxy / 301-redirect** to the new backend → stale-IP users
  still served correctly, zero broken experience.
- ✅ Cleanup = **DRAIN, don't YANK**: monitor traffic to old IP, decommission only when it hits ~0.
- PRINCIPLE: you don't own the client's DNS cache → old endpoint must keep working until
  nobody uses it. (Same idea reused in blue-green deploys & API versioning.)

---

## 🎬 "What happens when you type a URL" (THE screening answer)
1. DNS resolve (browser→OS→resolver→root→TLD→authoritative; Anycast→nearest edge).
2. **TCP** handshake (SYN→SYN-ACK→ACK).
3. **TLS** handshake (HTTPS key exchange).
4. **HTTP GET** sent.
5. **Load balancer (D2)** routes → server checks **cache (D3)** → DB on miss; **CDN (D4)** serves static.
6. Browser renders HTML, fetches CSS/JS/images (many from CDN).
- FINISH THE WHOLE LOOP — don't stop at "HTTP request"; include response + render.

---

## Personal Notes (recurring gaps)
- [ ] Prescribe the SPECIFIC ACTION (Q3: "lower the TTL in advance", not just "TTL caches it").
- [ ] FINISH THE FULL LOOP (Q4: include LB→cache→DB→render, not just up to HTTP request).
- [x] Naming discipline now consistent across the whole answer ✅.
