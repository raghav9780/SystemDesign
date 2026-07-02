# Day 13 — Proxies & Reverse Proxies
**Phase 1: Foundations** | Score: 9/10

> **One-sentence idea:** A proxy is a **middleman that sits between two parties and relays their traffic** —
> both sides talk to the proxy, not each other. The whole topic = **which side it protects**: a **forward
> proxy** fronts the *clients* (hides who they are); a **reverse proxy** fronts the *servers* (hides what's
> behind it). Same idea, opposite directions.
>
> Analogy: forward proxy = a personal assistant who makes all *your* calls (world sees the assistant, not you);
> reverse proxy = a **company receptionist** every visitor talks to, who routes them inside (visitors never see
> the org chart).

## 1. What a proxy is
A → **Proxy** → B. All traffic flows through it, so it can **observe / modify / cache / filter / redirect**.
One chokepoint = one place to enforce policy.

## 2. Forward proxy — fronts the CLIENTS
Client is **configured** to use it; it forwards on the client's behalf; destination sees the **proxy's IP**.
- **Corporate egress control** (block sites, log, enforce policy) · **caching** for a user group · **anonymity
  / bypass geo-blocks** (what a VPN essentially does). Key: acts for the client; the client knows it's there.

## 3. Reverse proxy — fronts the SERVERS ⭐
Clients think the reverse proxy **is** the server; it picks which backend handles each request; clients don't
know how many servers / IPs / tech exist. Jobs: load balancing · TLS termination · caching · compression ·
security (hide origins, WAF, DDoS) · request routing. Key: acts for the server; client is usually oblivious.

## 4. ⭐ Forward vs Reverse (crisp distinction)
| | **Forward** | **Reverse** |
|---|---|---|
| Fronts | **Clients** | **Servers** |
| Acts for | The client | The server |
| Who knows | The client (configured) | Usually nobody client-side |
| Hides | Client's identity from server | Servers' identity from client |
- One-liner: **"forward proxy hides the client from the server; reverse proxy hides the servers from the client."**

## 5. ⭐ Reverse proxy vs Load Balancer vs API Gateway
A spectrum of "intermediary in front of your servers," each more app-aware:
- **Load balancer** — one job: distribute traffic across identical servers (Day 2). A *feature*, usually
  provided by a reverse proxy.
- **Reverse proxy** — broader: load-balances PLUS TLS termination, caching, compression, routing; can front
  **even one server**. Every LB does reverse-proxy work; a reverse proxy does more.
- **API Gateway** (Day 12) — a **specialized reverse proxy** with app-layer smarts: auth, rate limiting,
  aggregation, versioning. Understands your *API*, not just HTTP.
- ⭐ Model: **Load Balancer ⊂ Reverse Proxy ⊂ API Gateway** (concentric). One tool (Nginx/Envoy) plays several roles.

## 6. Reverse-proxy job list (cold)
1. **SSL/TLS termination** — proxy holds the cert, decrypts HTTPS, forwards plain HTTP (backends need no certs).
   *(Variant: **TLS passthrough** — forward encrypted; backend terminates.)*
2. **Load balancing** (round-robin / least-conn / ip-hash, Day 2). 3. **Caching** (esp. static). 4. **Compression**
   (gzip/brotli). 5. **Static content serving**. 6. **Security** (hide origin IPs, WAF, DDoS, rate limiting).
7. **Request routing** (L7, by path / host header). 8. **Header manipulation** (`X-Forwarded-For`, strip headers).

## 7. ⭐ Service mesh & sidecar
- **North-south** = clients → system (edge reverse proxy/gateway). **East-west** = internal services calling
  each other — who does TLS/retries/metrics for those?
- **Sidecar pattern:** a small proxy **next to every service instance**; service calls go through its local
  sidecar; sidecars talk to each other. App code stays networking-dumb — sidecar handles **mTLS, retries+backoff,
  timeouts, circuit breaking, LB, observability** transparently.
- **Service mesh** = all sidecars (**data plane**, usually **Envoy**) + central **control plane** (**Istio**,
  **Linkerd**) configuring them → consistent security/reliability **without app code changes.**

## 8. Tools map
**Nginx** (canonical reverse proxy/web server/LB/cache/TLS) · **HAProxy** (high-perf LB, L4+L7) · **Envoy**
(service-mesh sidecar) · **Traefik** (auto discovery) · **Caddy** (auto HTTPS). CDN (Day 4) ≈ giant distributed
reverse proxy. AWS ALB / API Gateway = managed.

## 9. ⭐ Implementation — Nginx reverse-proxy config (line by line)
```nginx
upstream app_servers {              # (1) named pool → Nginx load-balances across it
    least_conn;                     #     algorithm (default round-robin); ip_hash also available
    server 10.0.0.1:8080;           #     NOTE: 'server' keyword + PORT are required
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
    server 10.0.0.4:8080;
    server 10.0.0.5:8080;
}
upstream static_server { server 10.0.0.9:8080; }

server {
    listen 443 ssl;                 # (2) 'listen' (NOT 'port') on 443 + enable TLS = TLS TERMINATION here
    server_name example.com;
    ssl_certificate     /etc/nginx/certs/example.com.crt;   # underscored directive names
    ssl_certificate_key /etc/nginx/certs/example.com.key;

    location /api/ {                 # (3) path-based (L7) routing — reads URL path, picks a pool
        proxy_pass http://app_servers;                      # relay as plain HTTP
        proxy_set_header Host              $host;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;   # (4) carry real client IP
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    location /images/ {
        proxy_pass http://static_server;                    # name MUST match the upstream exactly
        proxy_set_header Host            $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
- **`upstream`** = the backend pool; listing servers makes Nginx the load balancer.
- **`listen 443 ssl` + `ssl_certificate`** = TLS termination; everything forwarded is plain HTTP.
- **`location`** = path-based L7 routing; `proxy_pass` relays to the chosen pool.
- **`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for`** — once Nginx is the middleman the backend
  sees *Nginx's* IP; this header carries the **original client IP** so backend logs / rate-limiters / geo-checks
  are correct (`$proxy_add_x_forwarded_for` appends to any existing chain). ⭐ This "why" is the part I omitted.
- Map: `upstream`=LB · `listen 443 ssl`=TLS termination · `location`=routing · `proxy_set_header`=preserve client identity.

---

## 🎯 Tasks & my answers
**Task 1 — Design (7/7):**
1. Forward hides the client from the server; reverse hides the servers from the client. ✅
2. **SSL/TLS termination** — decrypt at the proxy, backends speak **plain HTTP** (no per-server certs). ✅
3. LB is a **subset** of reverse proxy; beyond balancing, Nginx also does TLS termination, caching, compression,
   rate limiting. ✅
4. **Path-based / content-based (L7) routing** — must be **L7** because only there can the proxy read the URL
   path/headers to decide the route (L4 sees only IP/port — ties to the Day 2 revision). ✅

**Task 2 — Nginx config (9/10):** structure all correct (two upstreams, 443+ssl+cert/key, two path `location`s,
`proxy_pass`). ⚠️ Lost 1 pt: set `X-Forwarded-For` but didn't state **why** the backend needs it (answer every
sub-part). Syntax fixes to remember: `server <ip>:<port>;`, `listen` not `port`, underscored directives,
`$proxy_add_x_forwarded_for` (exact name), and upstream names must match exactly.

---

## ⭐ Follow-up (RESOLVED)
**Q:** TLS terminates at Nginx, so Nginx→backend traffic is plain HTTP on the internal network. (1) Security
risk, and when? (2) In **zero-trust** (don't trust the internal network), what two options protect that hop, and
which Day 13 concept does it automatically for service-to-service traffic?

**(1) Yes — a risk whenever the internal network isn't trusted.** The hop is plaintext, so anyone who can tap the
wire (a compromised host, a rogue service, a mirrored switch port) reads credentials, tokens, and PII in the clear.
**Accepted** only on a genuinely isolated/trusted LAN or for low-sensitivity traffic, where the performance win of
skipping re-encryption outweighs the risk. *(Answered correctly.)*

**(2) Two options to protect the hop:**
1. **TLS re-encryption** — Nginx terminates the client's TLS, then opens a *fresh* TLS connection to the backend
   (`proxy_pass https://...`). Hop is encrypted again. Contrast **TLS passthrough** (Nginx never decrypts; backend
   terminates).
2. **mTLS (mutual TLS)** — *both* sides present certificates → not just encryption but mutual **authentication**.
   A rogue service can't connect without a valid cert. This is the zero-trust gold standard.

**The concept that does it automatically = service mesh + sidecar (§7).** The Envoy sidecar beside every service
performs **mTLS** on all east-west calls transparently; the control plane (Istio/Linkerd) issues and rotates the
certs — **no application code changes.** *(I named the service-mesh/sidecar answer correctly but skipped naming the
two options — the recurring "answer every sub-part" gap.)*

⭐ **One-liner:** "Protect the internal hop with TLS re-encryption or, better, mTLS for mutual auth — and a service
mesh gives you mTLS everywhere for free."

---

## Personal Notes (recurring gaps)
- [ ] **Answer EVERY sub-part:** Req 4 asked to set the header AND say why — I set it, skipped the why. Same gap
  as Day 8 Q3 / Day 9 Q5. When a question has two clauses, answer both.
- [ ] **Config grammar:** `server <ip>:<port>;`, `listen 443 ssl;`, underscored directives,
  `$proxy_add_x_forwarded_for`, exact upstream-name matches. Intent was right; grammar costs a real config.
- [x] **Naming much improved** ✅ — used "subset," "SSL/TLS termination," "L7" precisely (the recurring gap is closing).
- [x] **Reverse-proxy structure in config = solid** ✅ (upstream pool, TLS listener, path routing, header forwarding).
