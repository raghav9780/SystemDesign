# Important Questions — Daily Revision Log


**Q:** What is the difference between an L4 and an L7 load balancer? Give one concrete routing decision that L7 can make but L4 cannot — and explain why L4 can't.

**A:** An **L4 load balancer** operates at the transport layer (TCP/UDP). It only sees connection-level info — source/destination **IP address and port** — and forwards the packet stream without inspecting the payload. It's fast and protocol-agnostic but content-blind.

An **L7 load balancer** operates at the application layer. It terminates the connection and parses the actual request (e.g. the HTTP method, URL path, headers, cookies), so it can make **content-based routing** decisions.

Concrete example only L7 can do: route `/api/*` requests to the API server pool and `/images/*` to a static-asset pool, or route based on a cookie/`Host` header. **Why L4 can't:** it never decrypts or parses the payload — the HTTP path/headers live *inside* the TCP byte stream that L4 just forwards, so that information is simply invisible to it.
