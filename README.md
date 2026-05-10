# System Design: 30 Core Concepts

> **How to use this guide**
> Every concept is explained with a definition, a real-world analogy, and a concrete example drawn from **designing Twitter/X** — so every piece connects to the same system. At the end you'll find a full architecture diagram, an interview cheat sheet, 10 classic interview questions, and curated links for deeper reading.

![Concepts](https://img.shields.io/badge/concepts-30-blue)
![Interview Ready](https://img.shields.io/badge/interview-ready-green)
![Twitter/X Example](https://img.shields.io/badge/example-Twitter%2FX-black)

---

## Table of Contents

**Part 1 — Networking Foundations**
1. [Client-Server Architecture](#1-client-server-architecture)
2. [IP Address](#2-ip-address)
3. [DNS](#3-dns)
4. [Proxy / Reverse Proxy](#4-proxy--reverse-proxy)
5. [Latency](#5-latency)
6. [HTTP / HTTPS](#6-httphttps)

**Part 2 — API Layer**

7. [APIs](#7-apis)
8. [REST API](#8-rest-api)
9. [GraphQL](#9-graphql)

**Part 3 — Data Storage**

10. [Databases](#10-databases)
11. [SQL vs NoSQL](#11-sql-vs-nosql)
15. [Database Indexing](#15-database-indexing)
16. [Replication](#16-replication)
17. [Sharding](#17-sharding)
18. [Vertical Partitioning](#18-vertical-partitioning)
20. [Denormalization](#20-denormalization)

**Part 4 — Scaling**

12. [Vertical Scaling](#12-vertical-scaling)
13. [Horizontal Scaling](#13-horizontal-scaling)
14. [Load Balancers](#14-load-balancers)

**Part 5 — Performance**

19. [Caching](#19-caching)
22. [Blob Storage](#22-blob-storage)
23. [CDN](#23-cdn)

**Part 6 — Distributed Systems Theory**

21. [CAP Theorem](#21-cap-theorem)

**Part 7 — Real-Time & Async**

24. [WebSockets](#24-websockets)
25. [Webhooks](#25-webhooks)
27. [Message Queues](#27-message-queues)

**Part 8 — Architecture Patterns**

26. [Microservices](#26-microservices)
28. [Rate Limiting](#28-rate-limiting)
29. [API Gateways](#29-api-gateways)
30. [Idempotency](#30-idempotency)

**Closing**
- [Full Twitter/X Architecture Diagram](#full-twitterx-architecture-diagram)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [10 Classic Interview Questions](#10-classic-interview-questions)
- [External Resources](#external-resources)

---

## The Big Example

> Throughout this guide we are designing **Twitter/X** — a social platform where users post short messages (tweets), follow each other, see a personalized feed, and receive real-time notifications. At any given second Twitter handles ~6,000 tweets written and ~600,000 reads. Every one of the 30 concepts below is illustrated with exactly where and why it appears in this system.

---

## Part 1 — Networking Foundations

---

### 1. Client-Server Architecture

**What it is:** A model where one process (the *client*) requests resources, and another process (the *server*) provides them. The client initiates; the server responds. They communicate over a network using agreed-upon protocols.

**Analogy:** A restaurant. You (client) sit at a table and give your order to the waiter (network). The kitchen (server) prepares the food and sends it back. You never walk into the kitchen yourself.

**Twitter/X example:**
- Your mobile app is the client. When you open your feed, the app sends an HTTP request to `api.twitter.com`. Twitter's servers fetch your personalized timeline and return a JSON response. The app renders it. Neither side can work without the other — the app has no data; the server has no UI.
- Twitter's backend services also act as clients to *each other*: the Timeline Service calls the User Service to fetch profile info. Client-server is recursive inside a distributed system.

**Interview tips:**
- Distinguish *thin clients* (browser — logic on server) from *thick clients* (mobile app — logic on device).
- Peer-to-peer (BitTorrent, WebRTC) breaks this model: every node is both client and server simultaneously.

**Learn more:** [MDN — Client-Server overview](https://developer.mozilla.org/en-US/docs/Learn/Server-side/First_steps/Client-Server_overview)

---

### 2. IP Address

**What it is:** A numerical label assigned to every device on a network. IPv4 uses 32 bits (`192.168.1.1`); IPv6 uses 128 bits (`2001:0db8::1`). It is the postal address of the internet — without it, packets have nowhere to go.

**Analogy:** A house number on a street. The street name is the network; the house number is the device. Mail (packets) can only be delivered if both are correct.

**Twitter/X example:**
- Every server in Twitter's data centers has a private IP (e.g., `10.0.4.22`). These are invisible to users.
- The public-facing load balancer has a *Virtual IP (VIP)* — a single stable IP that all clients connect to (e.g., `104.244.42.1`). If a server fails and is replaced, the VIP stays the same; clients never notice.
- DNS maps `api.twitter.com` → this VIP. You never type the IP — but every connection ultimately uses one.

**Interview tips:**
- Know the difference between *public* IPs (routable on the internet) and *private* IPs (RFC 1918: `10.x`, `172.16–31.x`, `192.168.x`).
- NAT (Network Address Translation) lets thousands of devices share one public IP — important for understanding cloud networking.

**Learn more:** [Cloudflare — What is an IP address?](https://www.cloudflare.com/learning/dns/glossary/what-is-my-ip-address/)

---

### 3. DNS

**What it is:** The Domain Name System is the internet's phone book. It translates human-readable domain names (`twitter.com`) into machine-readable IP addresses (`104.244.42.1`). Without DNS, you'd have to memorize IPs for every website.

**Analogy:** A phone book. You look up "Twitter" and get the phone number (IP). You don't need to know the number in advance — just the name.

**Twitter/X example:**
- When you type `twitter.com`, your OS checks its local cache first. If not found, it asks a *recursive resolver* (usually your ISP or `8.8.8.8`). The resolver walks the DNS tree: root servers → `.com` nameservers → Twitter's authoritative nameservers → returns `104.244.42.1`.
- Twitter uses **low TTLs** (30–60 seconds) on its DNS records so they can quickly reroute traffic during a data center failure — change the IP record and within a minute all new connections go to the healthy DC.
- Twitter also uses **GeoDNS** to return different IPs for users in different regions, routing European users to European data centers for lower latency.

**Interview tips:**
- TTL (Time To Live) controls how long DNS responses are cached. Low TTL = faster failover but more DNS queries. High TTL = less load but slower changes.
- A/CNAME/MX records: `A` maps name → IPv4, `CNAME` aliases one name to another, `MX` specifies mail servers.

**Learn more:** [Cloudflare — How DNS works](https://www.cloudflare.com/learning/dns/what-is-dns/) | [ByteByteGo — DNS in depth](https://blog.bytebytego.com/p/a-closer-look-at-dns)

---

### 4. Proxy / Reverse Proxy

**What it is:**
- **Forward proxy:** Sits between clients and the internet. Clients send requests through it. Hides client identity. Used for filtering, privacy, corporate egress.
- **Reverse proxy:** Sits between the internet and your servers. Clients think they're talking to the server directly. Used for load balancing, SSL termination, caching, and security.

**Analogy:** A **forward proxy** is like a secretary who makes calls on your behalf — the recipient only knows the secretary's number. A **reverse proxy** is like the receptionist at a large office — all visitors go through them, never directly to the employee.

**Twitter/X example:**
- **Reverse proxy (Nginx/Envoy):** Every request to `api.twitter.com` hits an Nginx reverse proxy first. It: terminates TLS (decrypts HTTPS), routes to the correct upstream service (tweets vs. users vs. search), strips internal headers, and adds `X-Request-ID` for tracing.
- **Forward proxy:** Twitter's internal services call external APIs (payment processors, SMS gateways) through a corporate forward proxy. This centralizes egress, allows URL filtering, and gives one IP for whitelisting by external vendors.
- **Twitter's CDN edge** (Akamai/Fastly) is also effectively a reverse proxy — it caches static content (JS, CSS, images) close to users.

**Interview tips:**
- Reverse proxies are your first line of defence: they can absorb DDoS traffic, block bad IPs, and rate-limit before requests reach app servers.
- Don't confuse with a load balancer — reverse proxies can do load balancing, but also caching, SSL termination, and header manipulation.

**Learn more:** [Nginx — Reverse proxy guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/) | [Cloudflare — What is a reverse proxy?](https://www.cloudflare.com/learning/cdn/glossary/reverse-proxy/)

---

### 5. Latency

**What it is:** The time delay between a cause (sending a request) and its effect (receiving a response). Measured in milliseconds. Latency has multiple components: network round-trip time, processing time, queuing time, and storage I/O time.

**Analogy:** The time between flicking a light switch and the light turning on. Even at the speed of light, electrons and photons have travel time — and the bulb takes a moment to heat up.

**Twitter/X example:**
- Twitter's SLA for timeline loading is **< 200ms at P99** (99th percentile). That means even 99% of the slowest users should see their feed in under 200ms.
- Latency sources in the Twitter stack:
  - Network RTT from client to data center: ~20–80ms (depends on geography — CDN and GeoDNS help)
  - Redis cache lookup: ~0.5ms
  - Cassandra tweet lookup on cache miss: ~5–20ms
  - Serialization + JSON encoding: ~2–5ms
  - Total target: ~30–50ms server-side processing
- During a trending event (e.g., a World Cup goal), write throughput spikes 10×. Queuing latency in Kafka can spike from 5ms to 500ms if consumers can't keep up.

**Latency numbers every engineer should know:**

| Operation | Typical latency |
|-----------|----------------|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| Main memory (RAM) | 100 ns |
| SSD random read | 150 µs |
| Network within data center | 0.5 ms |
| Redis GET | 0.5–1 ms |
| Database query (indexed) | 1–10 ms |
| Cross-region network | 50–150 ms |
| HDD seek | 10 ms |

**Interview tips:**
- P50/P95/P99 percentiles matter more than averages. An average of 50ms can hide 1% of users experiencing 5 seconds.
- Latency vs. throughput: you can improve throughput (requests/sec) with parallelism, but latency is bounded by physics (speed of light).

**Learn more:** [Latency Numbers Every Programmer Should Know](https://gist.github.com/jboner/2841832) | [ByteByteGo — Latency numbers](https://blog.bytebytego.com/p/latency-numbers-every-programmer)

---

### 6. HTTP/HTTPS

**What it is:** HTTP (HyperText Transfer Protocol) is the foundation of data communication on the web — a request/response protocol over TCP. HTTPS adds TLS encryption so the data can't be intercepted or tampered with in transit.

**Analogy:** HTTP is a postcard — anyone who handles it can read it. HTTPS is a sealed, tamper-proof envelope — only the sender and recipient can read the contents.

**Twitter/X example:**
- All public Twitter API calls use HTTPS. Your browser sends: `GET /home HTTP/2` with headers including `Authorization: Bearer <token>`. Twitter's server responds with `200 OK` and a JSON body.
- **TLS termination** happens at Nginx (the reverse proxy). Nginx decrypts the HTTPS, then forwards plain HTTP to internal services — no need to manage certificates on every app server.
- Twitter uses **HTTP/2** for multiplexing: a single TCP connection carries multiple requests in parallel (your app fetches tweets, ads, trending topics simultaneously without waiting for each to finish).
- **HTTP/3** (QUIC protocol) is being rolled out on mobile where UDP reduces connection setup latency vs. TCP.

**HTTP methods summary:**

| Method | Meaning | Idempotent? | Safe? |
|--------|---------|------------|-------|
| GET | Fetch resource | Yes | Yes |
| POST | Create resource | No | No |
| PUT | Replace resource | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Remove resource | Yes | No |

**Interview tips:**
- HTTP status codes: 2xx (success), 3xx (redirect), 4xx (client error), 5xx (server error). Know 200, 201, 400, 401, 403, 404, 429, 500, 503.
- HTTPS is non-negotiable for any production system handling user data. Cookies should always have `Secure` and `HttpOnly` flags.

**Learn more:** [MDN — HTTP overview](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview) | [Cloudflare — What is HTTPS?](https://www.cloudflare.com/learning/ssl/what-is-https/)

---

## Part 2 — API Layer

---

### 7. APIs

**What it is:** An Application Programming Interface defines a contract between two software components: what requests can be made, in what format, and what responses to expect. APIs hide implementation details — you call the function without knowing how it works inside.

**Analogy:** A TV remote control. You press "Volume Up" and the TV gets louder. You don't need to know about the infrared signal encoding, the microcontroller, or the speaker circuit — the remote is the API that hides all that complexity.

**Twitter/X example:**
- Twitter exposes a **public API v2** that lets third-party apps read tweets, post on behalf of users, manage followers, and stream real-time data.
- Internally, the Tweet Service exposes an API to the Timeline Service: `getTweetsByIds(ids: string[]): Tweet[]`. The Timeline Service doesn't know or care whether tweets are stored in Cassandra or MySQL — it just calls the API.
- This is why APIs are the backbone of microservices: services communicate only through their published APIs, never by directly touching each other's databases.

**Types of APIs you should know:**
- **REST** — Stateless, resource-based, HTTP verbs (most common)
- **GraphQL** — Client specifies exact fields needed, single endpoint
- **gRPC** — Binary protocol, fast, used for internal service-to-service calls
- **WebSockets** — Persistent, bidirectional (covered in concept 24)
- **Webhooks** — Reverse API: server calls client (covered in concept 25)

**Interview tips:**
- API versioning matters: `v1`, `v2`, or date-based (`2024-01-01`). Never break existing clients.
- Document APIs with OpenAPI/Swagger so consumers can self-serve.

**Learn more:** [What is an API? (Cloudflare)](https://www.cloudflare.com/learning/security/api/what-is-an-api/)

---

### 8. REST API

**What it is:** Representational State Transfer is an architectural style for APIs built on HTTP. Key principles: resources identified by URLs, stateless requests (server keeps no session between calls), standard HTTP verbs (GET/POST/PUT/DELETE), and uniform interface.

**Analogy:** A filing cabinet with labelled drawers. Each drawer (URL) holds a specific type of document (resource). You open/add/remove/replace documents using standardized actions — you don't have a special procedure for each drawer.

**Twitter/X example:**

| Action | HTTP method | Endpoint | Body |
|--------|------------|----------|------|
| Get a tweet | GET | `/tweets/1234567890` | — |
| Post a tweet | POST | `/tweets` | `{"text": "Hello!"}` |
| Delete a tweet | DELETE | `/tweets/1234567890` | — |
| Follow a user | POST | `/users/me/following` | `{"target_user_id": "999"}` |
| Get my timeline | GET | `/timelines/home` | — |
| Update profile | PATCH | `/users/me` | `{"name": "New Name"}` |

**Statelessness in practice:** Each request includes the auth token (`Authorization: Bearer <token>`). The server doesn't remember who you are between requests — it re-validates the token every time. This makes horizontal scaling trivial: any server can handle any request.

**Interview tips:**
- REST is *architectural style*, not a protocol. There's no spec enforcement — "RESTful" is a spectrum.
- HATEOAS (Hypermedia As The Engine Of Application State): responses include links to next actions. Rarely implemented fully in practice.
- Pagination: use cursor-based (tweet_id as cursor) not offset-based (`?page=3`) for large, changing datasets — offset skips rows, causing missed items as new tweets arrive.

**Learn more:** [RESTful API Design best practices (ByteByteGo)](https://blog.bytebytego.com/p/restful-api-design-tips)

---

### 9. GraphQL

**What it is:** A query language for APIs where the *client* specifies exactly what data it needs in a single request. One endpoint (`/graphql`), flexible queries, and a typed schema. Avoids over-fetching (getting more fields than needed) and under-fetching (needing multiple requests to get all required data).

**Analogy:** Instead of ordering a fixed "Meal A" from the menu (REST — you get what the server decided), you tell the chef exactly what you want: "I'd like a burger but swap the bun for lettuce, no pickles, add bacon, and a side salad." You get exactly what you asked for — no more, no less.

**Twitter/X example:**
- A mobile client rendering a tweet card needs: tweet text, author name, author avatar URL, like count, and whether *I* have liked it. With REST you might call `/tweets/123` (gets the full tweet object with 40 fields) and then `/users/456` separately — that's two round trips and you get back 35 fields you don't need.
- With GraphQL, one request:
  ```graphql
  query {
    tweet(id: "123") {
      text
      author { name avatarUrl }
      likeCount
      likedByMe
    }
  }
  ```
- Twitter considered GraphQL for their mobile apps to reduce data usage on low-bandwidth connections.

**REST vs GraphQL — when to choose:**

| Consideration | REST | GraphQL |
|--------------|------|---------|
| Multiple clients with different data needs | Painful (multiple endpoints or over-fetch) | Excellent fit |
| Simple CRUD APIs | Clean and simple | Overkill |
| Caching | Easy (HTTP cache by URL) | Harder (all POST to `/graphql`) |
| File uploads | Simple | Complex |
| Team learning curve | Low | Medium |
| Real-time (subscriptions) | Needs separate WebSocket | Built-in subscriptions |

**Interview tips:**
- N+1 problem: a GraphQL query for 10 tweets that each need author info can trigger 10 separate DB calls. Solved with **DataLoader** (batching).
- Facebook invented GraphQL in 2012 and open-sourced it in 2015.

**Learn more:** [GraphQL official docs](https://graphql.org/learn/) | [Shopify on GraphQL vs REST](https://shopify.engineering/underpinnings-of-graphql)

---

## Part 3 — Data Storage

---

### 10. Databases

**What it is:** A structured system for storing, retrieving, and managing data. Databases provide durability (data survives crashes), consistency (rules are enforced), and efficient querying. Choosing the right database is one of the most impactful architectural decisions you make.

**Analogy:** Different types of storage for different things in a home — a filing cabinet for documents, a photo album for pictures, a whiteboard for temporary notes. No single storage system is optimal for everything.

**Twitter/X example — multiple databases for different jobs:**

| Data | Database | Why |
|------|---------|-----|
| User profiles, auth | MySQL (SQL) | Relational, transactional, strong consistency |
| Tweets | Cassandra (NoSQL) | Write-heavy, massive scale, time-series friendly |
| Home timeline (precomputed) | Redis | In-memory, sub-millisecond reads |
| Search index | Elasticsearch | Full-text search, relevance ranking |
| Media metadata | MySQL | Structured, foreign keys to users |
| Session tokens | Redis | TTL-based expiry, fast lookup |
| Analytics / metrics | ClickHouse | Column-oriented, fast aggregations |
| Graph (follows) | MySQL + custom graph store | Traversal queries |

**Interview tips:**
- No single database is best for everything. Production systems routinely use 3–5 different database technologies.
- ACID: Atomicity, Consistency, Isolation, Durability — the guarantee SQL databases provide for transactions.

**Learn more:** [Use the index, Luke (SQL tuning guide)](https://use-the-index-luke.com/) | [Designing Data-Intensive Applications — Martin Kleppmann (book)](https://dataintensive.net/)

---

### 11. SQL vs NoSQL

**What it is:**
- **SQL (relational):** Data in tables with rows and columns. Rigid schema. Supports JOINs and ACID transactions. Examples: MySQL, PostgreSQL, SQLite.
- **NoSQL:** Various non-relational models — document (MongoDB), key-value (Redis), wide-column (Cassandra), graph (Neo4j). Flexible schema, often sacrifice some ACID guarantees for scale and speed.

**Analogy:** SQL is a spreadsheet — structured rows and columns, rigid. NoSQL is a collection of Post-it notes — flexible, fast, unstructured, harder to cross-reference.

**Twitter/X example — why both are needed:**

The user table:
```
users (id, username, email, created_at, bio, location)
```
This is relational. Users have a *relationship* to followers, tweets, DMs. SQL's JOIN lets us query: "Give me all tweets liked by people I follow." That's a complex relational query — SQL is the right tool.

Tweets, however, are write-heavy (6,000/sec) and read-heavy (600,000/sec). A single SQL table with billions of rows and no horizontal scaling would buckle. **Cassandra** stores tweets as wide rows partitioned by `user_id`. No JOINs — you query by user ID, get back their tweets in time order. Fast writes, fast range reads, horizontally scalable.

**Decision guide:**

| Choose SQL when... | Choose NoSQL when... |
|-------------------|---------------------|
| Data is relational (JOINs needed) | Data is document-like or hierarchical |
| Strong consistency required | Eventual consistency is acceptable |
| Complex queries / aggregations | Simple key-value or range lookups |
| Schema is stable | Schema changes frequently |
| Transactions across multiple tables | Write throughput > millions/sec |

**Interview tips:**
- "NoSQL" doesn't mean "no schema" — it means the schema is enforced by the application, not the database. This can be a bug factory if undisciplined.
- Many modern SQL databases (PostgreSQL, MySQL 8) support JSON columns and can handle semi-structured data, blurring the line.

**Learn more:** [MongoDB — SQL vs NoSQL](https://www.mongodb.com/compare/relational-vs-non-relational-databases) | [ByteByteGo — SQL vs NoSQL](https://blog.bytebytego.com/p/understanding-database-types)

---

### 15. Database Indexing

**What it is:** An index is an additional data structure (usually a B-tree or hash) that maps column values to row locations, so the database can find rows without scanning the entire table. Reads become dramatically faster; writes become slightly slower (the index must be updated too).

**Analogy:** The index at the back of a textbook. Without it, you'd scan every page to find "Cassandra." With the index, you go directly to page 247. The book itself (data) didn't change — the index is extra metadata pointing into it.

**Twitter/X example:**

The tweets table has billions of rows. Without an index:
```sql
SELECT * FROM tweets WHERE user_id = 12345 ORDER BY created_at DESC LIMIT 20;
```
→ Full table scan. Scans all 50 billion rows. Takes minutes. Unusable.

With a **composite index on `(user_id, created_at)`**:
→ The database jumps directly to user 12345's block of rows, already sorted by time. Milliseconds.

**Covering index:** If the index includes all columns the query needs, the database never touches the main table at all:
```sql
-- Index: (user_id, created_at, tweet_id, text)
-- This query is served entirely from the index:
SELECT tweet_id, text FROM tweets WHERE user_id = 12345 ORDER BY created_at DESC LIMIT 20;
```

**Common index types:**

| Type | Use case | Example |
|------|---------|---------|
| B-tree | Range queries, equality, ORDER BY | `WHERE created_at BETWEEN x AND y` |
| Hash | Equality only, very fast | `WHERE session_id = 'abc'` |
| Full-text | Text search | `WHERE MATCH(text) AGAINST('hello')` |
| Composite | Multi-column filters | `WHERE user_id = X AND status = 'active'` |

**Interview tips:**
- Indexes speed reads but slow writes and use disk space. Don't index every column.
- Cardinality matters: indexing a boolean column (only 2 values) is almost useless. Indexing `user_id` (millions of values) is very useful.
- Use `EXPLAIN` to see if your query is using an index.

**Learn more:** [Use the Index, Luke!](https://use-the-index-luke.com/sql/anatomy) | [PostgreSQL index types](https://www.postgresql.org/docs/current/indexes-types.html)

---

### 16. Replication

**What it is:** Keeping copies of the same data on multiple machines. The primary node accepts writes; replicas sync and serve reads. Goals: fault tolerance (if primary dies, a replica takes over), read scalability (spread read load across replicas), and geographic distribution (replicas in different regions).

**Analogy:** Google Docs auto-saves to multiple data centers. If one burns down, your document is safe on the others. Any employee in any office can read the shared document (read scaling).

**Twitter/X example:**
- The MySQL user database uses **primary-replica replication**: one primary handles all writes (new users, profile updates, follows), 5 read replicas handle profile lookups.
- Twitter's global read traffic is ~80% reads. Replication lets them serve 4× more reads without touching the primary.
- **Async replication:** The primary writes and acknowledges the client immediately, then asynchronously syncs to replicas. ~10ms replica lag. Risk: if primary crashes before sync, that write is lost.
- **Sync replication:** Primary waits for at least one replica to confirm before acknowledging. Zero data loss, but higher latency. Twitter uses sync replication for financial/auth data; async for tweet content.

**Replication types:**

| Type | Pros | Cons |
|------|------|------|
| Single-primary | Simple, no conflicts | Primary is write bottleneck |
| Multi-primary | Write to any node | Conflict resolution complexity |
| Leaderless (Dynamo-style) | High availability | Eventual consistency |

**Interview tips:**
- Replication solves **availability** and **read scale** — not write scale. For write scale, you need sharding (concept 17).
- Replica lag is a subtle bug source: a user writes a tweet, then immediately reads their profile — if the read hits a replica that hasn't synced yet, they see stale data. Solution: route the user's own reads to primary for 1 second after a write (*read-your-own-writes consistency*).

**Learn more:** [Designing Data-Intensive Applications — Chapter 5](https://dataintensive.net/) | [ByteByteGo — Database Replication](https://blog.bytebytego.com/p/master-slave-replication)

---

### 17. Sharding

**What it is:** Splitting a single large database into multiple smaller *shards* (partitions), each holding a subset of the data. Each shard is a separate database server. Solves the problem of a single machine not having enough disk, RAM, or CPU to handle the full dataset.

**Analogy:** Imagine a library so large no single building can hold all books. You split by genre: one building for fiction, one for non-fiction, one for science. Each building (shard) can be staffed and expanded independently.

**Twitter/X example:**
- Tweets table has 500 billion rows. No single Cassandra node holds all of them. Shard key: `user_id % 1000` → 1,000 shards across hundreds of nodes.
- When the Timeline Service requests tweets for user `12345`: `12345 % 1000 = 345` → query shard 345 directly.
- **Hotspot problem:** Elon Musk has 170M followers. Every write to his shard is also fanned out to millions of follower timelines. The shard for user `12345` (Musk's shard) is a "hot shard" — it receives disproportionate write load. Solution: celebrity accounts use a special *fan-out-on-read* strategy (see concept 20).

**Sharding strategies:**

| Strategy | How | Risk |
|---------|-----|------|
| Hash-based | `hash(user_id) % N` | Adding shards requires resharding |
| Range-based | User IDs 1–1M on shard 1, 1M–2M on shard 2 | Uneven distribution if IDs cluster |
| Directory-based | Lookup table maps key → shard | Lookup table is a bottleneck |
| Consistent hashing | Virtual nodes on a ring | Complex but allows smooth shard addition |

**Interview tips:**
- Sharding makes cross-shard JOINs very hard. Avoid queries that need data from multiple shards — restructure data so most queries touch one shard.
- Resharding (changing the number of shards) is painful. Plan shard count for 2–3 years of growth. Consistent hashing minimizes resharding cost.

**Learn more:** [ByteByteGo — Database Sharding](https://blog.bytebytego.com/p/database-sharding) | [Cloudflare — What is database sharding?](https://www.cloudflare.com/learning/performance/what-is-sharding/)

---

### 18. Vertical Partitioning

**What it is:** Splitting a database table by *columns* rather than rows (rows are split by sharding). You move certain columns — or entire logical domains — into separate tables or separate services. Opposite of sharding (which splits rows across servers).

**Analogy:** Instead of one giant filing folder with personal details, medical history, financial records, and work history, you create four separate folders. Each is smaller, easier to secure, and can be accessed independently.

**Twitter/X example:**

Original monolithic `tweets` table:
```
tweets (id, user_id, text, created_at, like_count, retweet_count,
        media_urls, media_types, media_sizes, reply_to_id,
        quote_tweet_id, language, is_sensitive, geo_lat, geo_lon, ...)
```

After vertical partitioning:
- `tweet_core` (id, user_id, text, created_at) — queried on every timeline load
- `tweet_metrics` (tweet_id, like_count, retweet_count, reply_count) — updated frequently, separate write load
- `tweet_media` (tweet_id, media_url, type, width, height) — only fetched when rendering media
- `tweet_geo` (tweet_id, lat, lon) — only for geo-search feature

Benefits:
- `tweet_core` is narrower → more rows fit in a memory page → faster full scans and cache hits.
- `tweet_metrics` can be sharded and cached independently (like counts update without touching tweet text).
- Security: `tweet_geo` can have stricter access controls without affecting the core service.

**Interview tips:**
- Vertical partitioning often maps naturally to microservices: each service owns its columns/table (see concept 26).
- Be careful of "chatty" services that need to JOIN across partitions on every request — you've traded a fat table for multiple network calls.

---

### 20. Denormalization

**What it is:** Intentionally introducing redundancy into a database design — storing derived or duplicated data — to speed up reads at the cost of more complex writes. The opposite of normalization (which eliminates redundancy).

**Analogy:** A normalized grocery store stocks ingredients. To make a sandwich, you fetch bread from aisle 3, cheese from aisle 7, ham from aisle 5, and assemble it yourself. A denormalized deli counter pre-makes the sandwich for you — it's ready instantly, but you need a bigger display case and someone to keep the pre-made sandwiches fresh.

**Twitter/X example — the fan-out-on-write timeline:**

**Normalized (fan-out-on-read):** Store tweets once. On each timeline load, query "tweets from all users I follow": 
```sql
SELECT t.* FROM tweets t
JOIN follows f ON t.user_id = f.followee_id
WHERE f.follower_id = :me
ORDER BY t.created_at DESC LIMIT 20
```
If you follow 1,000 people, this touches ~1,000 rows in the follows table and potentially millions of tweet rows. At 600,000 timeline loads/second → catastrophic.

**Denormalized (fan-out-on-write):** When a tweet is posted, immediately write it into the pre-built timeline of every follower:
```
Redis key: timeline:{follower_id}
Value: sorted set of tweet_ids, ordered by time
```
Reading the timeline is now a single Redis `ZREVRANGE` call — ~0.5ms. The cost: on every tweet, you write to thousands of follower timeline caches. For a user with 100 followers that's fine. For Elon Musk (170M followers), a single tweet triggers 170M writes — hence the hybrid approach for celebrities.

**When to denormalize:**
- Read:write ratio is very high (timelines are read 100× more than written)
- The normalized JOIN query is too slow at scale
- You can tolerate slight eventual consistency

**Interview tips:**
- Denormalization is a performance optimization, not a data model improvement. Only do it when you have a measured performance problem.
- You must keep denormalized copies in sync. If a tweet is deleted, remove it from all timeline caches — a new failure mode.

---

## Part 4 — Scaling

---

### 12. Vertical Scaling

**What it is:** Making a single machine more powerful — more CPU cores, more RAM, faster SSDs. Also called "scaling up." Simplest form of scaling: no code changes needed, just upgrade the hardware.

**Analogy:** A taxi driver who handles more passengers by buying a bigger van. Same driver, same route, just more capacity.

**Twitter/X example:**
- Early Twitter (2006–2008) ran on a single MySQL server. As traffic grew, they kept buying bigger servers: more RAM to cache indexes, faster CPUs, NVMe SSDs. This worked until it didn't.
- The 2008 "Fail Whale" era: even the biggest available servers couldn't handle Super Bowl traffic. Vertical scaling had hit its ceiling (~$500K/server, ~4TB RAM, ~100+ cores was the practical max at the time).
- They were forced to horizontally scale — a painful two-year re-architecture.

**Limits of vertical scaling:**
- Physical ceiling: there's a maximum size machine you can buy
- Cost curve is non-linear: 2× RAM often costs 4× money
- Single point of failure: one server dies, everything is down
- Downtime required for most hardware upgrades

**When vertical scaling is still the right call:**
- Database primary servers (vertical scale before horizontal to delay resharding pain)
- Applications with inherently stateful, hard-to-parallelize workloads
- When the operational simplicity of one server is worth the cost

**Learn more:** [Vertical vs Horizontal Scaling (DigitalOcean)](https://www.digitalocean.com/resources/article/horizontal-scaling-vs-vertical-scaling)

---

### 13. Horizontal Scaling

**What it is:** Adding more machines and distributing the load across them. Also called "scaling out." Enables near-unlimited scale in theory. Requires your application to be *stateless* — any instance can handle any request.

**Analogy:** Instead of one large van, you hire a fleet of small taxis and dispatch customers to whichever one is free. Adding a car is cheap; coordinating the fleet adds complexity.

**Twitter/X example:**
- Twitter's tweet-serving layer runs thousands of stateless pods in Kubernetes. During a live sporting event, tweet read traffic can spike 50× in seconds. Kubernetes auto-scales from 200 pods to 2,000 pods in ~3 minutes.
- **Statelessness is the prerequisite.** No session data lives in the app server. The auth token goes in the request header, session state lives in Redis, and user data in the DB. Any pod can handle any request — the load balancer can freely distribute.
- **Database horizontal scaling is harder than app-layer scaling.** You can add web servers in minutes. Adding a database shard requires careful data migration and resharding — which is why Twitter spent years doing it.

**Horizontal scaling checklist:**
- [ ] Application is stateless (no local file system state, no in-memory session)
- [ ] Shared state externalized (Redis, DB, distributed cache)
- [ ] Health checks in place (load balancer needs to know which instances are healthy)
- [ ] Deployment is automated (manual deploys don't scale to 2,000 pods)
- [ ] Logs/metrics aggregated centrally (you can't SSH into 2,000 servers)

**Learn more:** [Designing for Scalability (AWS)](https://docs.aws.amazon.com/whitepapers/latest/aws-overview/horizontal-and-vertical-scalability.html)

---

### 14. Load Balancers

**What it is:** A component that distributes incoming traffic across multiple backend servers. Acts as a single entry point; hides the fleet of servers behind it. Does health checks and stops routing to unhealthy instances.

**Analogy:** A supermarket checkout manager who watches all the queues and says "Register 5 is open — please move over!" They ensure no single cashier is swamped while others are idle.

**Twitter/X example:**
- All requests to `api.twitter.com` hit an AWS Network Load Balancer first. It distributes ~600,000 requests/sec across ~2,000 API pods using **least-connections** routing (route to the pod currently handling the fewest active requests).
- During a deploy, Twitter does **rolling updates**: 10% of pods are taken out, updated, health-checked, then returned. The load balancer never routes to a pod that fails its health check (`GET /health` returns 200). Zero-downtime deploy.
- **L4 vs L7 load balancers:**
  - L4 (TCP/UDP level): fast, simple, doesn't look at HTTP content. Good for raw throughput.
  - L7 (HTTP level): can route based on URL path (`/api/*` → API pods, `/stream/*` → WebSocket pods), can add headers, can sticky-session by user ID.

**Load balancing algorithms:**

| Algorithm | How | Best for |
|-----------|-----|---------|
| Round-robin | Pod 1, 2, 3, 1, 2, 3... | Uniform request cost |
| Least connections | Route to pod with fewest active requests | Variable request duration |
| IP hash | Hash client IP → same pod always | When session stickiness needed |
| Weighted round-robin | Heavier pods get more traffic | Heterogeneous fleet |

**Interview tips:**
- A load balancer is itself a potential SPOF (single point of failure). Production setups run load balancers in active-active pairs.
- Health checks are critical. A pod that's up but returning 500 errors should be removed from rotation.

**Learn more:** [What is a load balancer? (Cloudflare)](https://www.cloudflare.com/learning/performance/what-is-load-balancing/) | [ByteByteGo — Load balancers](https://blog.bytebytego.com/p/load-balancing-algorithms)

---

## Part 5 — Performance

---

### 19. Caching

**What it is:** Storing the results of expensive operations (DB queries, API calls, computation) in a fast, temporary store so future requests can be served without repeating the work. The core principle: compute once, serve many times.

**Analogy:** A chef's mise en place — pre-chopped vegetables and pre-measured spices ready at the station. The first customer waits while the chef preps; every subsequent customer gets their dish in seconds. The prep work (cache population) is the cost; the fast service is the payoff.

**Twitter/X example — caching layers (outer to inner):**

```
Browser cache
    ↓ (miss)
CDN cache (Fastly edge node, ~50ms from user)
    ↓ (miss)
API gateway cache (hot queries cached for 5 sec)
    ↓ (miss)
Redis cache (timeline:{user_id} → list of tweet IDs, TTL 30s)
    ↓ (miss)
Cassandra (source of truth — slow, ~10ms)
```

- **Timeline cache:** Redis stores the top 800 tweet IDs for each user's home timeline. A timeline request hits Redis first — ~0.5ms. Only on a cache miss does it fall through to Cassandra.
- **Cache invalidation:** When you post a new tweet, the Timeline Service removes your tweet ID from your followers' Redis caches (cache eviction), forcing a Cassandra read on their next timeline load. This is the hard part.

**Cache eviction policies:**
- **LRU (Least Recently Used):** Evict the key not accessed for the longest time. Good for timelines.
- **LFU (Least Frequently Used):** Evict the key accessed least often. Good for trending content.
- **TTL (Time To Live):** Every key expires after N seconds. Simple, prevents stale data accumulation.

**Cache patterns:**
- **Cache-aside:** App checks cache; on miss, reads DB and populates cache. Most common.
- **Write-through:** App writes to cache and DB simultaneously. Cache always consistent with DB.
- **Write-behind:** App writes to cache; async worker flushes to DB. Fast writes, risk of data loss on crash.

**Interview tips:**
- **Cache stampede:** All keys expire simultaneously; thousands of requests hit the DB at once. Solution: randomize TTL (30s ± 10s random jitter), or use probabilistic early expiration.
- Cache should store *expensive operations*, not cheap ones. Caching a primary key lookup that takes 1ms isn't worth the complexity.

**Learn more:** [Caching strategies (AWS)](https://aws.amazon.com/caching/best-practices/) | [ByteByteGo — Caching](https://blog.bytebytego.com/p/top-caching-strategies)

---

### 22. Blob Storage

**What it is:** Object (blob) storage is a system for storing large, unstructured binary data — images, videos, audio, documents. Files are stored as objects with a unique key, not in a file-system hierarchy. Highly durable, infinitely scalable, cheap. Examples: AWS S3, Google Cloud Storage, Azure Blob Storage.

**Analogy:** A self-storage facility with numbered units. You rent a unit, get a key (URL), and store whatever you want. You don't care what building layout organizes it internally — you just use your key to retrieve your stuff.

**Twitter/X example:**
- Every image and video in a tweet is stored in Twitter's object store (S3-compatible).
- **Upload flow:** Your app → Twitter API → Tweet Service generates a presigned S3 URL → your app uploads the file directly to S3 (bypasses Twitter's API servers, saves bandwidth) → S3 stores the file → Tweet is created with the S3 object key.
- **Presigned URLs:** A time-limited URL that grants temporary permission to upload or download a specific object without exposing credentials. Expires after 15 minutes.
- **Video at Twitter:** Videos are uploaded to S3, then an async transcoding job converts them to multiple resolutions (360p, 720p, 1080p) and stores each variant in S3. The CDN serves the appropriate variant based on the viewer's connection speed.

**Key properties of blob storage:**
- Durability: typically 99.999999999% (11 nines) — files are replicated across 3+ availability zones
- Eventual consistency on overwrite (new uploads are eventually consistent)
- Not designed for partial file writes (must upload the entire object)
- Much cheaper per GB than SSDs ($0.023/GB/month on S3 vs ~$0.10/GB for SSD)

**Interview tips:**
- Never store blobs in a relational database (as BLOB columns). This kills DB performance and negates the durability/cost benefits of object storage.
- Use CDN in front of object storage for read-heavy media (concept 23) — never serve media directly from S3 at scale.

**Learn more:** [AWS S3 documentation](https://docs.aws.amazon.com/s3/index.html) | [Cloudflare R2 vs S3](https://www.cloudflare.com/products/r2/)

---

### 23. CDN

**What it is:** A Content Delivery Network is a geographically distributed network of edge servers that cache content close to end users. Reduces latency by serving content from a node near the user instead of a distant origin server. Also absorbs traffic load and mitigates DDoS.

**Analogy:** Costco has warehouses all over the country so products are close to their customers. Without CDN, every customer drives to the single HQ in Seattle. With CDN, there's a warehouse 30 minutes from everyone.

**Twitter/X example:**
- **Static assets:** Twitter's JavaScript, CSS, and UI images are served from CDN (Akamai/Fastly) edge nodes. A user in Mumbai gets the app assets from an edge node in Chennai — ~5ms vs ~200ms from a US data center.
- **Tweet media:** Images and videos in tweets are cached at CDN edges. The first user to view a viral tweet causes a CDN miss (fetches from S3). Every subsequent viewer gets it from the CDN edge — S3 serves thousands of requests; CDN serves hundreds of millions.
- **Cache-Control headers:** Twitter sets `Cache-Control: max-age=86400` on profile images (cache for 24h) and `Cache-Control: max-age=31536000, immutable` on versioned JS bundles (cache forever — the URL changes when the file changes). 
- **CDN purge:** When a user changes their avatar, Twitter calls the CDN API to invalidate (purge) the cached old image from all edge nodes. Without this, users would see the old avatar for 24 hours.

**CDN use cases:**
- Static assets (JS, CSS, images)
- Video streaming (HLS/DASH segments)
- API responses (GET endpoints with cache headers)
- Large file downloads
- DDoS protection (absorbs attack traffic at the edge before it reaches origin)

**Interview tips:**
- CDN is not just for media — API responses with `Cache-Control` headers can be CDN-cached too. "What time does this store open?" doesn't need to hit your DB on every request.
- CDN providers: Cloudflare (also a reverse proxy/WAF), Fastly (programmable with VCL/Compute@Edge), Akamai (enterprise), AWS CloudFront.

**Learn more:** [What is a CDN? (Cloudflare)](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/)

---

## Part 6 — Distributed Systems Theory

---

### 21. CAP Theorem

**What it is:** In a distributed system, you can only guarantee **two of three** properties simultaneously:
- **C**onsistency: Every read returns the most recent write (or an error)
- **A**vailability: Every request receives a response (not necessarily the most recent data)
- **P**artition tolerance: The system continues operating even when network partitions split nodes

The catch: **network partitions will happen** in any real distributed system. So the real choice is **CP or AP** — how do you behave when the network breaks?

**Analogy:** A bank with two branches that share a ledger over a phone line:
- **CP:** If the phone line breaks, both branches stop accepting transactions until communication is restored. Consistent, but unavailable.
- **AP:** If the phone line breaks, both branches keep operating independently. Available, but they may both accept conflicting transactions — inconsistent until they reconnect and merge.

**Twitter/X example:**

| Subsystem | Choice | Reasoning |
|-----------|--------|-----------|
| Home timeline (Redis/Cassandra) | **AP** | Better to show a slightly stale feed than an error. 30s lag is acceptable. |
| Tweet counters (likes, RTs) | **AP** | A like count that's off by 3 for 2 seconds is fine. |
| User authentication | **CP** | Logging in must check the actual current auth state. An AP auth system could allow a deactivated account to log in. |
| Payment / Twitter Blue subscription | **CP** | Never charge twice. Consistency is non-negotiable. |
| DNS | **AP** | DNS keeps serving cached responses even during network issues. You might get a stale IP, but you won't get a timeout. |
| Your bank's ATM | **CP** | The ATM would rather go offline than let you overdraft twice. |

**Eventual Consistency:** Most AP systems guarantee *eventual* consistency — given no new writes, all replicas will *eventually* converge to the same value. This is the practical implementation of AP: a tweet's like count on replica A says 1,203 and replica B says 1,198 — within 2 seconds, both converge to 1,205 (the correct count).

**Interview tips:**
- "CA" (Consistent + Available but not Partition-tolerant) is theoretically possible in a single-node system, but single nodes fail. Distributed systems must tolerate partitions.
- PACELC extends CAP: even without a partition, there's a tradeoff between latency (L) and consistency (C) — answer PACELC in senior interviews.

**Learn more:** [CAP Theorem (Martin Fowler)](https://martinfowler.com/bliki/TwoPhaseCommit.html) | [Illustrated CAP Theorem](https://mwhittaker.github.io/blog/an_illustrated_proof_of_the_cap_theorem/) | [Designing Data-Intensive Applications — Chapter 9](https://dataintensive.net/)

---

## Part 7 — Real-Time & Async

---

### 24. WebSockets

**What it is:** A protocol that establishes a persistent, bidirectional, full-duplex connection between client and server over a single TCP connection. Unlike HTTP (client always initiates, server responds and closes), WebSockets allow the server to push data to the client at any time.

**Analogy:** HTTP is like sending a letter and waiting for a reply — one letter, one reply, connection closed. WebSockets is like a phone call — the line stays open, and either side can speak at any time.

**Twitter/X example:**
- **TweetDeck** (Twitter's power-user dashboard) shows live columns of tweets in real time. The browser opens a WebSocket connection to `streaming.twitter.com`. As new tweets matching your filter are posted, the server pushes them instantly — no polling.
- **Notifications:** The notification bell updates in real time. Instead of your browser polling `GET /notifications` every 5 seconds (100 requests/min per user × millions of users = catastrophic), one WebSocket connection carries all notifications for the life of your session.
- **Lifecycle:**
  1. Client upgrades HTTP connection: `Upgrade: websocket` header
  2. Server responds `101 Switching Protocols`
  3. Both sides can now send "frames" at any time
  4. Either side can close with a close frame

**WebSocket vs HTTP polling:**

| Aspect | HTTP Long-polling | WebSocket |
|--------|------------------|-----------|
| Connection overhead | New TCP handshake per poll | One TCP connection |
| Latency | Up to poll interval delay | Near-zero (server pushes immediately) |
| Server connections | Many short-lived | Fewer, long-lived |
| Load balancer complexity | Simple | Needs sticky sessions or pub/sub |

**Interview tips:**
- WebSocket connections are stateful — the connection is pinned to one server. For horizontal scaling, use a pub/sub layer (Redis Pub/Sub, Kafka) so any server can receive an event and fan it out to connected clients.
- Fallback: if WebSockets aren't supported, Server-Sent Events (SSE) provide one-way push with plain HTTP. Simpler, but unidirectional.

**Learn more:** [WebSockets explained (Ably)](https://ably.com/topic/websockets) | [MDN WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)

---

### 25. Webhooks

**What it is:** A reverse API — instead of your application polling a server for new events, the server calls your application's endpoint when an event happens. You register a URL; the provider POSTs event data to it when something occurs.

**Analogy:** Instead of calling a restaurant every 5 minutes to ask "Is my table ready?", you give them your phone number and they call you when it's ready. You go about your day; they initiate contact when it matters.

**Twitter/X example:**
- **Twitter Account Activity API:** A developer builds a customer support bot for a brand account. Instead of polling `GET /mentions` every 30 seconds, they register a webhook: `POST https://mybrand.com/twitter-events`. When anyone mentions the brand, Twitter POSTs the event JSON within milliseconds to that URL.
- Payload example:
  ```json
  {
    "tweet_create_events": [{
      "id": "123",
      "text": "@Brand your service is broken!",
      "user": { "screen_name": "angry_customer" }
    }]
  }
  ```
- **Webhook security:** Twitter signs each payload with HMAC-SHA256 using a shared secret. Your server validates the signature before processing — this prevents attackers from sending fake events to your webhook URL.

**Webhook reliability challenges:**
- Your server might be down when the event fires → provider should retry with exponential backoff
- Your server might be slow → provider times out → duplicate events on retry → your handler must be **idempotent** (concept 30)
- Event ordering is not guaranteed → use event timestamps, not arrival order

**Webhooks vs polling vs WebSockets:**
- **Polling:** Your server asks "any new events?" repeatedly. Simple but wasteful.
- **Webhooks:** Provider tells you when something happens. Efficient, but your server must be publicly accessible.
- **WebSockets:** Long-lived connection, bidirectional. Best for user-facing real-time UI (browser-to-server).

**Learn more:** [Webhook.site (test webhooks)](https://webhook.site) | [Stripe Webhooks documentation](https://stripe.com/docs/webhooks)

---

### 27. Message Queues

**What it is:** A component that enables asynchronous communication between services. The *producer* sends a message to the queue and immediately continues. The *consumer* reads from the queue at its own pace and processes it. Services are decoupled — producers don't wait for consumers, and a slow consumer doesn't block the producer.

**Analogy:** A restaurant order ticket system. The waiter (producer) writes the order on a ticket and clips it to the wheel (queue). The chef (consumer) grabs tickets and cooks them independently. The waiter doesn't stand at the kitchen window waiting — they seat more customers. If the kitchen gets backed up, tickets queue up, but the dining room still runs.

**Twitter/X example — the fan-out pipeline:**

```
User posts tweet
    ↓
Tweet Service → writes to Cassandra
    ↓
Tweet Service → publishes event to Kafka topic: tweet_created
    ↓
Fan-out Service (Kafka consumer) reads event
    ↓
Fetches follower list for that user (from User Service)
    ↓
For each follower: writes tweet_id to their Redis timeline cache
    ↓
Notification Service (another Kafka consumer) reads same event
    ↓
Sends push notifications to followers with notifications enabled
    ↓
Search Indexing Service (another Kafka consumer) reads same event
    ↓
Indexes tweet text in Elasticsearch
```

**One Kafka event → three completely independent consumer groups → three different systems updated**, all asynchronously, all without the Tweet Service knowing or waiting.

**Kafka key concepts:**
- **Topic:** Named channel for messages (e.g., `tweet_created`)
- **Partition:** Topics split into partitions for parallel consumption; messages within a partition are ordered
- **Consumer group:** Multiple consumers sharing a topic — each partition consumed by one consumer in the group
- **Retention:** Kafka keeps messages for a configured period (e.g., 7 days) — consumers can re-read past events

**Queue vs Streaming:**

| | Message Queue (SQS, RabbitMQ) | Event Streaming (Kafka) |
|-|-------------------------------|------------------------|
| Message deleted after read | Yes | No (configurable retention) |
| Multiple consumers of same message | No (one consumer wins) | Yes (consumer groups) |
| Ordering | Per-queue | Per-partition |
| Replay past events | No | Yes |

**Interview tips:**
- Message queues are the primary tool for decoupling microservices and handling traffic spikes (queue absorbs burst, consumers process at steady rate).
- At-least-once delivery: queues guarantee delivery but may deliver duplicates. Consumers must handle duplicates idempotently (concept 30).

**Learn more:** [Kafka in 5 minutes (Confluent)](https://kafka.apache.org/intro) | [AWS SQS vs SNS vs Kafka](https://aws.amazon.com/compare/the-difference-between-sqs-and-sns/)

---

## Part 8 — Architecture Patterns

---

### 26. Microservices

**What it is:** An architectural approach where an application is built as a collection of small, independently deployable services. Each service owns a single business capability and its own database. Services communicate over APIs (REST, gRPC) or message queues. Contrast with a *monolith* — a single codebase deployed as one unit.

**Analogy:** A Swiss army knife (monolith) vs. a toolbox (microservices). The knife is convenient — one thing, always together. The toolbox is flexible — replace the hammer without touching the screwdriver, use the right tool for each job. But you do need a box to carry them.

**Twitter/X example:**

| Service | Responsibility | Owns |
|---------|---------------|------|
| Tweet Service | Create/read/delete tweets | Cassandra tweets table |
| User Service | Profiles, auth, follows | MySQL users + follows tables |
| Timeline Service | Build/read home timelines | Redis timeline caches |
| Notification Service | Push/email/in-app notifications | Notification DB |
| Search Service | Index and query tweet content | Elasticsearch |
| Media Service | Upload, transcode, serve media | S3 + media metadata DB |
| Ads Service | Ad targeting and delivery | Ads DB |

Each service:
- Runs as independent containers (deployed on Kubernetes)
- Has its own CI/CD pipeline — can deploy without touching other services
- Scales independently — Search can be scaled without touching Notifications
- Fails independently — if the Ads Service is down, timelines still work

**Monolith vs Microservices:**

| | Monolith | Microservices |
|-|---------|--------------|
| Deployment | One unit | Many independent units |
| Scaling | Must scale everything | Scale individual services |
| Team ownership | Shared codebase friction | Clear service ownership |
| Latency | In-process calls (fast) | Network calls (slower) |
| Complexity | Simple to start | Complex infrastructure |
| Right for | Early stage, small team | Scale, large org, clear domains |

**Interview tips:**
- Start with a monolith. Extract microservices when you have a clear boundary with different scaling needs or team ownership. Premature microservices are an anti-pattern.
- The hardest part of microservices is distributed transactions: updating two services' databases atomically. Use the Saga pattern or accept eventual consistency.

**Learn more:** [Microservices.io patterns](https://microservices.io/patterns/index.html) | [Martin Fowler — Microservices](https://martinfowler.com/articles/microservices.html)

---

### 28. Rate Limiting

**What it is:** Controlling how many requests a client (user, IP, API key) can make in a given time window. Protects services from being overwhelmed, prevents abuse, and ensures fair resource distribution.

**Analogy:** A nightclub with a velvet rope. The bouncer lets in a certain number of people per hour. Even if a thousand people show up at once, only the allowed quota gets in — the rest wait or are turned away. The club's service quality stays high for those inside.

**Twitter/X example:**
- Twitter API v2 rate limits: `GET /tweets/search/recent` → **450 requests per 15 minutes** per app token. Exceed it and you get `HTTP 429 Too Many Requests` with a `Retry-After: 837` header (seconds until reset).
- **User-level rate limiting:** Each user can post max 300 tweets/day. A bot trying to spam 10,000 tweets/day is blocked after 300.
- Rate limits are applied at the **API Gateway** (concept 29) before requests reach application servers — so no load reaches the backend from abusive clients.

**Rate limiting algorithms:**

| Algorithm | How it works | Pros | Cons |
|-----------|-------------|------|------|
| Fixed Window | Count requests per fixed time period (e.g., per minute). Reset at :00, :01, :02... | Simple | Burst allowed at window boundary (100 req in last sec of minute + 100 req in first sec of next = 200 req in 2 sec) |
| Sliding Window Log | Keep a log of all request timestamps. Count how many in the last N seconds. | Accurate, no boundary burst | High memory (store all timestamps) |
| Sliding Window Counter | Blend fixed window counts weighted by how much of the window has passed. | Memory efficient, accurate | Approximation |
| Token Bucket | Bucket refills at a steady rate. Each request consumes a token. Burst allowed if tokens accumulated. | Allows bursts, smooth | Harder to implement |
| Leaky Bucket | Requests enter a queue; processed at a fixed rate. Like water leaking from a bucket. | Very smooth output | Not burst-friendly |

Twitter uses a **token bucket** variant: you accumulate credits over time, and bursty usage drains them — good for APIs that should allow short bursts but not sustained abuse.

**Interview tips:**
- For distributed rate limiting (multiple API Gateway nodes), use a shared Redis counter. `INCR` and `EXPIRE` atomically. Lua scripts for compare-and-set.
- Always return `Retry-After` header so well-behaved clients back off correctly.

**Learn more:** [Rate limiting patterns (Cloudflare)](https://www.cloudflare.com/learning/bots/what-is-rate-limiting/) | [System Design — Rate Limiter (ByteByteGo)](https://blog.bytebytego.com/p/rate-limiting-fundamentals)

---

### 29. API Gateways

**What it is:** A single entry point for all client requests to a backend system. The gateway sits in front of all microservices and handles cross-cutting concerns: authentication, rate limiting, request routing, SSL termination, logging, monitoring, and protocol translation. Clients talk to the gateway; the gateway talks to microservices.

**Analogy:** An airport security checkpoint. No matter which terminal you're flying from, everyone goes through security first. Security handles: identity verification (auth), prohibited items (request validation), directs you to the right terminal (routing), and logs who passed (observability). The airlines (microservices) don't each run their own security — it's centralized.

**Twitter/X example:**

```
Mobile app / Browser
    ↓ HTTPS
API Gateway (Kong / AWS API Gateway / Envoy)
    │  - Validate JWT token
    │  - Check rate limits (300 tweets/15min for this token)
    │  - Route by path:
    │      /tweets/* → Tweet Service
    │      /users/*  → User Service  
    │      /search/* → Search Service
    │      /stream/* → WebSocket Gateway
    │  - Add X-User-ID, X-Request-ID headers
    │  - Log request start
    │  - Record latency metrics
    ↓
Appropriate microservice
```

**Without an API Gateway:** every microservice would need to implement its own auth, rate limiting, and logging — massive duplication, inconsistent behavior, security gaps.

**API Gateway vs Load Balancer vs Reverse Proxy:**

| Component | Primary job |
|-----------|------------|
| Load Balancer | Distribute traffic across identical instances (L4/L7) |
| Reverse Proxy | Forward client requests to servers; SSL termination |
| API Gateway | L7 routing + auth + rate limiting + observability |

In practice, these overlap — Nginx can act as all three. Kong and AWS API Gateway are purpose-built for the gateway role.

**Interview tips:**
- The gateway must be highly available — it's a single point of failure for the entire platform. Deploy in active-active pairs across availability zones.
- GraphQL often uses a "schema stitching" API gateway that federates multiple downstream GraphQL services into one schema.

**Learn more:** [Kong API Gateway docs](https://docs.konghq.com/) | [AWS API Gateway overview](https://aws.amazon.com/api-gateway/)

---

### 30. Idempotency

**What it is:** An operation is idempotent if performing it multiple times produces the same result as performing it once. Crucial in distributed systems where network failures cause retries — you need to ensure retrying a request doesn't double-charge, duplicate records, or corrupt state.

**Analogy:** Pressing an elevator button. Pressing "Floor 5" once or ten times both result in the elevator going to floor 5. The extra presses have no additional effect. Contrast with a vending machine — pressing "Dispense" twice gives you two snacks.

**Twitter/X example:**

**Problem:** Your mobile app posts a tweet. The request reaches the server, the tweet is created, but the network drops before the `201 Created` response arrives. Your app doesn't know if it succeeded. It retries. Without idempotency: two identical tweets posted.

**Solution — idempotency key:** The client generates a unique key for the operation (e.g., `uuid4()`). Sends it in the header: `Idempotency-Key: a3b4c5d6-...`. The server stores the key + result in Redis (TTL 24h). On retry, the server finds the existing key and returns the cached result — no second tweet is created.

```
POST /tweets
Idempotency-Key: 7f3a-8e2b-...
Body: {"text": "Hello!"}

→ First call:  creates tweet, stores key→result in Redis, returns 201
→ Retry call:  finds key in Redis, returns same 201 with same tweet ID
→ No duplicate
```

**HTTP method idempotency:**
- `GET`: Always idempotent (read-only, no side effects)
- `PUT`: Idempotent — "set resource to this state" — same result no matter how many times called
- `DELETE`: Idempotent — first call deletes, subsequent calls return 404 (resource already gone — same end state: resource doesn't exist)
- `POST`: **Not idempotent** by default — each call creates a new resource. Must add idempotency key to make it safe to retry.
- `PATCH`: Not idempotent — `{"increment_likes": 1}` called twice increments twice

**Message queue idempotency:** Kafka and SQS guarantee *at-least-once delivery* — a message may be delivered twice. Your consumer must be idempotent: check if this event was already processed before acting on it.

**Interview tips:**
- Every payment operation must be idempotent. Stripe, Braintree, and Adyen all require idempotency keys on charge endpoints.
- Idempotency is a critical property for any retry logic — and all production systems retry.

**Learn more:** [Stripe — Idempotent requests](https://stripe.com/docs/api/idempotent_requests) | [AWS — Idempotency in distributed systems](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)

---

## Full Twitter/X Architecture Diagram

This diagram shows how all 30 concepts connect in the complete Twitter/X system:

```
                         ┌─────────────────────────────────────────────────────────┐
                         │                    CLIENTS (1)                          │
                         │  Mobile App          Browser          Third-party App   │
                         └──────────────────────────┬──────────────────────────────┘
                                                    │ HTTPS (6)
                                                    ▼
                         ┌──────────────────────────────────────────────────────────┐
                         │             DNS (3) → resolves twitter.com               │
                         │             ↓                                            │
                         │             CDN (23) — static assets, media              │
                         │             ↓ (on miss)                                  │
                         │      Load Balancer / Reverse Proxy (4, 14)               │
                         └──────────────────────────┬───────────────────────────────┘
                                                    │
                                                    ▼
                         ┌──────────────────────────────────────────────────────────┐
                         │              API Gateway (29)                            │
                         │  • Auth validation      • Rate Limiting (28)            │
                         │  • Routing              • Logging / Tracing             │
                         └─────┬───────────┬────────────┬─────────────┬────────────┘
                               │           │            │             │
               REST API (7,8)  │  GraphQL  │  gRPC      │  WebSocket  │ (24)
                               ▼           ▼            ▼             ▼
          ┌────────────────────────────────────────────────────────────────────────┐
          │                    MICROSERVICES (26)                                  │
          │                                                                        │
          │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
          │  │ Tweet Service│  │ User Service │  │Timeline Svc  │                │
          │  │  (REST API)  │  │  (REST API)  │  │  (REST API)  │                │
          │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                │
          │         │                 │                  │                        │
          │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
          │  │Search Service│  │  Notif. Svc  │  │ Media Service│                │
          │  │(Elasticsearch│  │  (Webhook 25)│  │  (Blob 22)   │                │
          │  └──────────────┘  └──────────────┘  └──────────────┘                │
          └──────────────────────────┬─────────────────────────────────────────────┘
                                     │
                                     ▼
          ┌──────────────────────────────────────────────────────────────────────────┐
          │                    MESSAGE QUEUE (27) — Kafka                           │
          │  Topics: tweet_created, like_created, follow_created, media_uploaded    │
          └──────────────┬─────────────────┬───────────────────┬────────────────────┘
                         │                 │                   │
                         ▼                 ▼                   ▼
                  Fan-out Service    Search Indexer      Notification Sender
                         │
                         ▼
          ┌──────────────────────────────────────────────────────────────────────────┐
          │                    DATA LAYER                                           │
          │                                                                        │
          │  Caching (19)              Databases (10)                             │
          │  ┌──────────────┐          ┌──────────────┐  ┌──────────────┐        │
          │  │ Redis         │          │  Cassandra   │  │  MySQL       │        │
          │  │ • Timelines  │          │  • Tweets    │  │  • Users     │        │
          │  │ • Sessions   │          │  • Sharded   │  │  • Follows   │        │
          │  │ • Counters   │          │    (17) by   │  │  • Replicated│        │
          │  │ • Rate limits│          │    user_id   │  │    (16)      │        │
          │  └──────────────┘          │  • Indexed   │  │  • Indexed   │        │
          │                            │    (15)      │  │    (15)      │        │
          │                            │  • Vert. Part│  └──────────────┘        │
          │                            │    (18)      │                           │
          │                            │  NoSQL (11)  │                           │
          │                            └──────────────┘                           │
          │                                                                        │
          │  Blob Storage (22): S3 — images, videos, audio                        │
          └──────────────────────────────────────────────────────────────────────────┘

CAP Theorem (21): Timeline/tweets → AP; Auth/payments → CP
Denormalization (20): Fan-out-on-write populates Redis timelines
Vertical/Horizontal Scaling (12,13): App layer scales horizontally; DBs scaled vertically first, then sharded
Latency (5): CDN + Redis ensure P99 < 200ms
Idempotency (30): All POST endpoints use idempotency keys; Kafka consumers deduplicate
```

---

## Interview Cheat Sheet

One sentence for "What is X?" — memorize these for quick definitions.

| # | Concept | One-liner |
|---|---------|-----------|
| 1 | Client-Server Architecture | Clients request, servers respond — a model where computation and data are centralized on servers and accessed by many clients |
| 2 | IP Address | A unique numerical label that identifies a device on a network, like a postal address for packets |
| 3 | DNS | The internet's phone book — maps human-readable domain names to machine-readable IP addresses |
| 4 | Proxy / Reverse Proxy | A forward proxy hides client identity; a reverse proxy sits in front of servers to route, cache, and terminate TLS |
| 5 | Latency | The time delay between sending a request and receiving a response, measured in milliseconds at percentiles (P50/P99) |
| 6 | HTTP/HTTPS | The request/response protocol of the web; HTTPS adds TLS encryption to prevent eavesdropping |
| 7 | APIs | A defined contract for how software components interact — hiding internal implementation behind a stable interface |
| 8 | REST API | A stateless, HTTP-based style where resources are identified by URLs and manipulated with standard verbs (GET/POST/PUT/DELETE) |
| 9 | GraphQL | A query language where the client specifies exactly what fields it needs, served through a single typed endpoint |
| 10 | Databases | Systems for storing, retrieving, and managing data with durability guarantees and efficient query support |
| 11 | SQL vs NoSQL | SQL: relational tables, ACID transactions, JOINs; NoSQL: flexible schema, horizontally scalable, trades some consistency for speed |
| 12 | Vertical Scaling | Making one machine more powerful (more CPU/RAM) — simple but hits a physical ceiling |
| 13 | Horizontal Scaling | Adding more machines and distributing load — requires stateless design but scales nearly infinitely |
| 14 | Load Balancers | Distribute traffic across a fleet of servers, perform health checks, and hide individual server failures from clients |
| 15 | Database Indexing | A secondary data structure that maps column values to row locations, enabling fast lookups without full table scans |
| 16 | Replication | Keeping synchronized copies of data on multiple machines for fault tolerance and read scalability |
| 17 | Sharding | Splitting data across multiple database machines by a shard key, enabling horizontal scaling of writes and storage |
| 18 | Vertical Partitioning | Splitting a table by columns (or domain) into separate tables/services, making each leaner and independently scalable |
| 19 | Caching | Storing results of expensive operations in a fast store (Redis) so repeat requests are served without hitting the source |
| 20 | Denormalization | Introducing intentional redundancy (duplicated or precomputed data) to speed reads at the cost of more complex writes |
| 21 | CAP Theorem | A distributed system can guarantee only 2 of 3: Consistency, Availability, Partition Tolerance — and partitions always happen |
| 22 | Blob Storage | Object storage for large unstructured files (images, videos) — infinitely scalable, durable, cheap per GB (e.g., S3) |
| 23 | CDN | A global network of edge servers that cache content close to users, reducing latency and offloading origin servers |
| 24 | WebSockets | A persistent bidirectional TCP connection that lets servers push data to clients instantly without polling |
| 25 | Webhooks | A reverse API where a provider POSTs event data to your endpoint when something happens, instead of you polling |
| 26 | Microservices | An architecture of small, independently deployable services each owning one capability and its own database |
| 27 | Message Queues | Async communication via a broker (Kafka, SQS) — producers write events, consumers process them independently and at their own pace |
| 28 | Rate Limiting | Controlling request volume per client per time window to prevent abuse and ensure fair resource distribution |
| 29 | API Gateways | A single entry point that handles auth, routing, rate limiting, and observability for all backend microservices |
| 30 | Idempotency | An operation that produces the same result whether executed once or many times — essential for safe retries |

---

## 10 Classic Interview Questions

Practice designing these systems using the 30 concepts above. Concept tags show which ones are most relevant.

---

**Q1. Design a URL Shortener (bit.ly)**
Key concepts: `[REST API]` `[Databases]` `[Caching]` `[Horizontal Scaling]` `[Load Balancers]` `[DNS]`
- Write: POST /shorten → generate short code → store in SQL (code → long URL)
- Read: GET /:code → lookup in Redis cache → fallback to DB → HTTP 301 redirect
- Scale: reads vastly outnumber writes → cache aggressively; shard by short code prefix

---

**Q2. Design a Messaging App (WhatsApp)**
Key concepts: `[WebSockets]` `[Message Queues]` `[Databases]` `[Replication]` `[Blob Storage]` `[CAP Theorem]`
- Messages delivered via WebSocket for online users; queued in Kafka for offline users
- Messages stored in Cassandra (write-heavy, time-series); media in S3
- CAP: choose AP — messages may be delivered slightly out of order, but never dropped

---

**Q3. Design a Ride-Sharing App (Uber)**
Key concepts: `[WebSockets]` `[Sharding]` `[Load Balancers]` `[Caching]` `[Message Queues]` `[Microservices]`
- Driver location updates every 4 seconds via WebSocket → stored in Redis geo-index
- Matching service: find nearest available driver using Redis `GEORADIUS`
- Shard by city/region — drivers in London never interact with drivers in Tokyo

---

**Q4. Design a Video Streaming Service (YouTube)**
Key concepts: `[Blob Storage]` `[CDN]` `[Message Queues]` `[Databases]` `[Vertical Partitioning]` `[Caching]`
- Upload: video → S3 → Kafka event → transcoding workers → multiple resolution variants → S3
- Stream: CDN serves HLS/DASH segments; adaptive bitrate selects quality by bandwidth
- Metadata (title, description, views) in MySQL; video files in S3; comments in Cassandra

---

**Q5. Design a Rate Limiter**
Key concepts: `[Rate Limiting]` `[Caching]` `[API Gateways]` `[Horizontal Scaling]`
- Token bucket stored in Redis: `INCR user:123:requests`, `EXPIRE user:123:requests 60`
- At API Gateway layer — no request reaches microservices if rate limited
- Distributed: all API Gateway nodes share Redis — no per-node counters that get out of sync

---

**Q6. Design a Notification System**
Key concepts: `[Message Queues]` `[Webhooks]` `[WebSockets]` `[Microservices]` `[Databases]`
- Event sources (tweet, like, follow) publish to Kafka
- Notification Service reads Kafka, fans out to: push (FCM/APNs), email (SendGrid), in-app (WebSocket)
- Deduplication: idempotency key on each notification so retries don't double-notify

---

**Q7. Design a Search Engine (Twitter Search)**
Key concepts: `[Databases]` `[Message Queues]` `[Horizontal Scaling]` `[Caching]` `[Sharding]`
- Index built from tweet_created Kafka events → Elasticsearch cluster
- Query: user types → autocomplete from Redis → full search hits Elasticsearch
- Shard Elasticsearch index by time (recent shards get more queries — cache them)

---

**Q8. Design a Distributed Cache (Redis-like)**
Key concepts: `[Caching]` `[Consistent Hashing]` `[Replication]` `[CAP Theorem]`
- Consistent hashing to distribute keys across nodes — minimize resharding when nodes added/removed
- Each key on primary node + 1 replica (AP: serve stale reads if primary fails)
- Eviction: LRU when memory full

---

**Q9. Design a Payment System**
Key concepts: `[Idempotency]` `[Databases]` `[Message Queues]` `[CAP Theorem]` `[Microservices]`
- All charge endpoints require idempotency key — retries never double-charge
- CAP: choose CP — better to fail than to charge twice
- Saga pattern for distributed transactions: order service + payment service + inventory service

---

**Q10. Design a News Feed (Facebook/Instagram)**
Key concepts: `[Denormalization]` `[Caching]` `[Message Queues]` `[Sharding]` `[Databases]` `[CDN]`
- Fan-out-on-write: on post, push to all followers' feed caches (Redis sorted sets)
- Hybrid for celebrities: fan-out-on-read to avoid 10M cache writes per post
- Media in S3 + CDN; feed ordering by score (engagement + recency)

---

## External Resources

Curated links — these replace long explanations here and are kept up-to-date by their maintainers.

### Essential References

| Resource | What it covers | Link |
|----------|---------------|------|
| System Design Primer | Comprehensive overview of every concept, with diagrams | [GitHub](https://github.com/donnemartin/system-design-primer) |
| ByteByteGo Newsletter | Visual system design explainers, interview-focused | [bytebytego.com](https://bytebytego.com) |
| Designing Data-Intensive Applications | Deep technical book on databases, distributed systems | [dataintensive.net](https://dataintensive.net) |
| Cloudflare Learning Center | Networking, CDN, DNS, security — all free | [cloudflare.com/learning](https://www.cloudflare.com/learning/) |
| High Scalability Blog | Real architecture case studies (Twitter, Netflix, Uber) | [highscalability.com](http://highscalability.com) |
| Martin Fowler's Blog | Microservices, patterns, distributed systems essays | [martinfowler.com](https://martinfowler.com) |
| AWS Architecture Blog | Real-world patterns on AWS | [aws.amazon.com/blogs/architecture](https://aws.amazon.com/blogs/architecture/) |

### Per-Concept Deep Dives

| Concept | Best resource |
|---------|--------------|
| DNS internals | [How DNS works (Julia Evans zine)](https://jvns.ca/blog/2022/01/11/how-to-learn-about-networking/) |
| HTTP/2 & HTTP/3 | [HTTP/3 explained (Cloudflare)](https://blog.cloudflare.com/http3-the-past-present-and-future/) |
| REST API design | [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines) |
| GraphQL | [GraphQL official learn](https://graphql.org/learn/) |
| SQL indexing | [Use the Index, Luke!](https://use-the-index-luke.com/) |
| CAP Theorem proof | [An illustrated proof of CAP](https://mwhittaker.github.io/blog/an_illustrated_proof_of_the_cap_theorem/) |
| Kafka architecture | [Kafka docs — introduction](https://kafka.apache.org/intro) |
| Consistent hashing | [ByteByteGo — Consistent Hashing](https://blog.bytebytego.com/p/consistent-hashing) |
| Microservice patterns | [microservices.io](https://microservices.io/patterns/index.html) |
| Rate limiting algorithms | [Rate limiting (Figma eng blog)](https://www.figma.com/blog/an-alternative-approach-to-rate-limiting/) |
| Idempotency | [Stripe idempotency docs](https://stripe.com/docs/api/idempotent_requests) |
| WebSockets | [Ably — WebSocket deep dive](https://ably.com/topic/websockets) |
| Load balancing algorithms | [AWS — ELB algorithms](https://aws.amazon.com/elasticloadbalancing/) |
| Database replication | [DDIA Chapter 5 (free excerpt)](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/) |

---

*All 30 concepts illustrated through a single Twitter/X system design. Every request you make on Twitter touches at least 15 of these concepts simultaneously.*
