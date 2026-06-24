# Important Questions — Daily Revision Log


**Q:** What is the difference between an L4 and an L7 load balancer? Give one concrete routing decision that L7 can make but L4 cannot — and explain why L4 can't.

**A:** An **L4 load balancer** operates at the transport layer (TCP/UDP). It only sees connection-level info — source/destination **IP address and port** — and forwards the packet stream without inspecting the payload. It's fast and protocol-agnostic but content-blind.

An **L7 load balancer** operates at the application layer. It terminates the connection and parses the actual request (e.g. the HTTP method, URL path, headers, cookies), so it can make **content-based routing** decisions.

Concrete example only L7 can do: route `/api/*` requests to the API server pool and `/images/*` to a static-asset pool, or route based on a cookie/`Host` header. **Why L4 can't:** it never decrypts or parses the payload — the HTTP path/headers live *inside* the TCP byte stream that L4 just forwards, so that information is simply invisible to it.

---

**Q:** CAP says that during a network partition you must choose between Consistency and Availability. PACELC extends CAP with a second clause that applies even when there is no partition (the "Else / Latency / Consistency" part). What trade-off does that clause describe, and give an example of a system tuned each way?

**A:** PACELC = "if **P**artition → choose **A**vailability or **C**onsistency; **E**lse (normal operation) → choose **L**atency or **C**onsistency."

The "Else" clause says: even with a perfectly healthy network (no partition), there is *still* a trade-off — to keep replicas **strongly consistent** you must wait for them to coordinate/agree on every read or write, which **adds latency**. So you can sacrifice some consistency to get lower latency, or pay latency to stay consistent.

Examples: **DynamoDB / Cassandra are PA/EL** — they favor availability during a partition and **low latency** during normal operation (eventual consistency). **Google Spanner is PC/EC** — it favors **consistency** in both states, accepting higher latency (it waits on quorum + TrueTime). A single-leader SQL DB with synchronous replication is also EC.
