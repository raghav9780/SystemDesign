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
  / bypass geo-blocks** (similar in effect to a VPN, though a VPN tunnels all traffic at the network layer). Key: acts for the client; the client knows it's there.

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

---

## ⭐ Interview-completeness additions (audit pass)

### A. Kubernetes Ingress + Ingress Controller — the modern "how do external requests reach my services?"
The container-era version of §3's reverse proxy. Two things, easy to conflate:
- **Ingress** = a **declarative rule object** (YAML). It just says the *what*: "host `api.foo.com`, path `/pay` → service `payments`." It's config, not a running process.
- **Ingress Controller** = the actual **reverse-proxy pod** that *reads* those rules and does the work: **L7 routing + TLS termination**. Commonly **Nginx Ingress**, or Envoy-based like **Contour**.
- ⭐ One-liner: **Ingress = the config; Ingress Controller = the reverse proxy enforcing it.** This is **north-south edge routing** (§7) for containerized systems — the same job as an Nginx `server`+`location` block, expressed as Kubernetes objects.

### B. When a service mesh is NOT worth it (§7's cost side — interviewers always ask this)
A mesh isn't free, and naming the cost is the senior signal:
- **Latency** — every sidecar adds a **network hop on every call** (in + out through the local proxy).
- **Resources** — Istio's own benchmark (1.24): a sidecar handling **1,000 req/s** uses about **~0.2 vCPU / ~60 MB
  RAM**, and it grows with traffic — multiplied across every pod. *(Corrected from an earlier "~0.5 vCPU / 50 MB"
  rule of thumb; source: https://istio.io/latest/docs/ops/deployment/performance-and-scalability/)*
- **Operational complexity** — control-plane upgrades, cert rotation, and debugging **two hops** per request.
- **Verdict:** adopt only at **meaningful microservice scale**, where consistent **mTLS / retries / observability** pays for itself. For a handful of services, a **library** (in-process) or a **plain reverse proxy** is simpler and cheaper.
- ⭐ Interview move: **name the cost, not just the benefit** — "I'd hold off on a mesh until we have enough services that consistent mTLS/observability outweighs a sidecar's per-call hop and its CPU/memory cost in every pod."

### C. WAF / DDoS — actually unpacked (§3/§6 listed these as bare words)
The reverse proxy is the natural **chokepoint** to enforce both once, for everything behind it.
- **WAF (Web Application Firewall)** — inspects **L7 request content** and blocks known **attack patterns** (**SQL injection, XSS, path traversal**) *before* they reach the app. **Rule-based** — e.g. the **OWASP Core Rule Set (CRS)**.
- **DDoS mitigation** — the proxy/edge **absorbs volumetric floods** via **per-IP rate limiting**, **connection limits**, **SYN-flood protection**, and **Anycast** to spread load across **PoPs** (ties to Day 4's CDN/edge).
- ⭐ Why here: one middleman = one place to enforce security policy for all backends (the §1 chokepoint principle applied to attacks).

### D. (brief) Nginx event-driven vs Apache thread-per-connection
- **Nginx** uses an **event-driven / async** model: a few **worker processes** each handle **thousands of connections** via an **event loop**.
- **Apache** (classic) uses **thread/process-per-connection** — one worker tied up per open connection.
- ⭐ Why it matters: Nginx handles **high concurrency (the C10k problem)** with a **small, stable memory footprint** — the reason it became the canonical reverse proxy.

### E. (brief) Caveat the concentric model (§5)
"**Load Balancer ⊂ Reverse Proxy ⊂ API Gateway**" is a great **teaching** model, but it's not a strict mathematical superset — an API gateway is a **SPECIALIZED reverse proxy** (adds app-layer smarts), not everything a reverse proxy is plus more.
- ⭐ In an interview, **lead with "an API gateway is a specialized reverse proxy"** and use the concentric picture only as intuition.

---

## ⭐ Completeness additions (pass 2)

### F. Forward proxy — how it actually works and where you'd meet one
**Intuition:** §2 said "the client is configured to use it." Concretely that means the browser/OS/app is told
"send every outbound request to `proxy.corp.com:3128` instead of straight to the internet" — via a browser setting,
a **PAC file** (a small script that says which URLs go through which proxy), or env vars like `HTTP_PROXY` /
`HTTPS_PROXY` for servers and CLIs.

**What about HTTPS?** The proxy can't read encrypted traffic. The client sends an HTTP **`CONNECT host:443`**
request; the proxy opens a TCP tunnel to that host and just shovels encrypted bytes both ways. So by default the
proxy sees **which site** you're going to, not **what** you send. Corporate proxies that need to inspect content do
**TLS interception**: they install a company root certificate on every managed laptop so the proxy can decrypt —
which is exactly why that's only done on devices the company controls.

**Use cases (name the category, then the example):**
1. **Egress control / content filtering** — block categories of sites, enforce acceptable-use policy (school/corp).
2. **Audit logging** — record who accessed what, for security/compliance.
3. **Outbound caching** — one office downloads the same OS update once, not 500 times (Squid is the classic tool).
4. **Anonymity / geo** — the destination sees the proxy's IP, not yours.
5. **Server egress in the cloud** — backend servers in a private subnet reach the internet **only** through an
   egress proxy that allow-lists destinations (e.g. only `api.stripe.com`), so a compromised server can't phone
   home to an attacker. This is the one most relevant to system-design interviews.

**Transparent proxy** = a forward proxy the client *doesn't* know about — the network silently routes traffic
through it. (Hold that thought — it's exactly how a sidecar captures traffic, below.)

⭐ One-liner: "A forward proxy is outbound control for clients — filtering, logging, caching, and locking down
which external hosts our servers may call."

### G. Sidecar mechanics — how does the app's traffic end up in Envoy without code changes?
**Intuition:** the app thinks it's calling `http://payments:8080` directly. It has no idea a proxy exists. So
something below the app must **hijack** the connection. That something is the Linux firewall, **iptables**.

How it works in Istio's sidecar mode (verified against Istio docs):
1. **Shared network namespace.** In Kubernetes, all containers in a pod share one network stack — same IP, same
   `localhost`. So the Envoy container and the app container see the same network traffic.
2. **iptables redirect.** Before the app starts, an init container (`istio-init`, which needs `NET_ADMIN`/`NET_RAW`
   capabilities) — or the **Istio CNI plugin**, which avoids giving pods those privileges — writes iptables rules
   in the pod: **all outbound TCP → Envoy on port 15001; all inbound TCP → Envoy on port 15006.**
3. **Loop prevention.** Envoy's own outgoing traffic must *not* be redirected back into itself, so Envoy runs as a
   reserved user ID (**UID 1337**) that the rules skip. (That's why Istio tells apps not to run as UID 1337.)
4. **Flow:** app → (iptables) → local Envoy → mTLS → remote pod's Envoy → (iptables) → remote app. Two proxy hops
   per call — the latency cost in §B.

Source: https://istio.io/latest/docs/ops/deployment/application-requirements/ (ports 15001/15006, UID 1337,
istio-init capabilities, CNI alternative)

⭐ One-liner: "The sidecar shares the pod's network namespace and iptables rules transparently redirect all
in/out traffic through it — that's why the app needs zero code changes."

### H. Control plane vs data plane — who does what
Same split as an airline: **pilots (data plane)** fly every plane; **air-traffic control (control plane)** tells
them where to go but never touches the controls.
- **Data plane = the proxies (Envoy sidecars).** They sit **in the request path** and do the per-request work:
  route, load-balance, mTLS encrypt/decrypt, retry, timeout, circuit-break, emit metrics/traces.
- **Control plane = `istiod` in Istio.** **Not in the request path.** It (1) does **service discovery** (watches
  Kubernetes for which pods back which service), (2) turns your high-level config (routing rules, canary weights,
  policies) into Envoy config and **pushes it to every proxy** (via Envoy's xDS APIs), and (3) acts as the
  **certificate authority** — issuing and rotating the mTLS certificates each proxy uses.
  *(Historically Istio split this into Pilot / Citadel / Galley; they were merged into the single `istiod` binary
  in Istio 1.5.)*
- **Why the split matters:** because the control plane isn't in the request path, if `istiod` goes down the
  proxies generally keep serving traffic with their last-known config — you just can't push changes (and certs
  can't be renewed) until it's back.

⭐ One-liner: "Data plane = Envoy proxies doing per-request work; control plane = istiod doing discovery, config
push and cert issuance, out of the request path."

### I. 2025-26 note — sidecar-less meshes (Istio ambient mode)
The sidecar's cost (§B: a proxy in every pod, two hops per call, restart pods to upgrade the proxy) pushed Istio to
a **sidecar-less** option: **ambient mode**, which reached **General Availability in Istio 1.24 (Nov 2024)**.
- **ztunnel** — one small **per-node** proxy (written in **Rust**) handling **L4** only: mTLS, identity, L4
  authorization, telemetry. Every pod on the node gets mTLS without its own sidecar.
- **waypoint proxy** — an **optional**, Envoy-based **L7** proxy deployed per namespace/service, only where you
  need L7 features (HTTP routing, retries, L7 authorization).
- **Why:** you pay for L7 only where you use it. Istio's benchmark at 1,000 req/s: a ztunnel used ~0.06 vCPU /
  ~12 MB vs a sidecar's ~0.2 vCPU / ~60 MB (and ztunnel is per *node*, not per *pod*).
- Sidecar and ambient workloads can coexist in one mesh.
Sources: https://istio.io/latest/blog/2024/ambient-reaches-ga/ · https://istio.io/latest/docs/ambient/overview/

⭐ Interview move: "In 2025-26 I'd also consider a sidecar-less mesh like Istio ambient — per-node L4 mTLS, with L7
proxies only where needed — which removes most of the per-pod sidecar overhead."

### J. Linkerd — the other mesh to name
Istio uses Envoy (general-purpose, feature-rich). **Linkerd** uses its own **`linkerd2-proxy`**, an ultralight
**Rust "micro-proxy"** built only for the service-mesh job (not a general-purpose proxy). Trade-off: Linkerd =
simpler, lighter, fewer knobs; Istio + Envoy = more features and extensibility, more complexity. Both give
automatic mTLS, retries and metrics.
Source: https://linkerd.io/2/reference/architecture/

### K. Summary table (after the prose)
| Concept | Lives where | Job | Example |
|---|---|---|---|
| Forward proxy | in front of clients | outbound filtering/logging/caching/egress allow-list | Squid, corp proxy |
| Reverse proxy | in front of servers | TLS termination, LB, caching, routing | Nginx, HAProxy |
| Sidecar | next to each app in the pod | mTLS, retries, metrics, transparently via iptables | Envoy, linkerd2-proxy |
| Data plane | every proxy | per-request work | Envoy sidecars / ztunnel |
| Control plane | central | discovery, config push, cert issuance | istiod |
| Ambient (sidecar-less) | per node (+ optional L7 waypoints) | mesh without per-pod sidecars | Istio ambient |
