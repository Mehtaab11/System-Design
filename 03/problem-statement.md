# TinyURL — Problem Statement

## 1. Problem Statement

Design a URL-shortening service similar to TinyURL that converts long URLs into short, unique URLs.

For example:

```text
Long URL:
https://example.com/products/category/some-very-long-product-url?id=12345

Short URL:
https://tiny.url/aB7xK92p
```

The system should allow users to:

1. Create a short URL from a long URL.
2. Redirect users from the short URL to the original long URL.
3. Handle a very large number of redirects efficiently.
4. Generate unique short codes with minimal collision.
5. Scale horizontally as traffic increases.

The most important characteristic of this system is that **URL reads are significantly more frequent than URL writes**.

For example, we can assume a **1:100 write-to-read ratio**.

If the system receives:

```text
1 billion URL creations / month
```

it could potentially receive:

```text
100 billion URL redirects / month
```

Therefore, the architecture should primarily be optimized for **high read throughput and low-latency redirects**.

---

# 2. Short URL Capacity

Suppose our short URL contains exactly 8 characters:

```text
tiny.url/xxxxxxxx
```

There are several choices for the character set.

### Numeric characters

If we use only:

```text
0-9
```

we have 10 possible characters.

Therefore:

```text
10⁸ = 100,000,000
```

possible combinations.

This is relatively small for a large-scale URL-shortening service.

### Alphanumeric characters

Instead, we can use:

```text
0-9
a-z
A-Z
```

This gives us:

```text
62 possible characters
```

Therefore:

```text
62⁸ = 218,340,105,584,896
```

That is approximately:

```text
218 trillion
```

possible unique short URLs.

This provides a very large namespace and is more than sufficient for a URL-shortening service operating at very high scale.

---

# 3. Back-of-the-Envelope Storage Estimation

Assume the service creates:

```text
1 billion URLs / month
```

For a simplified estimation, suppose each database record contains:

```text
Long URL       ≈ 100 bytes
Metadata       ≈ 50 bytes
```

Therefore:

```text
Total record size ≈ 150 bytes
```

For 1 billion records:

```text
150 bytes × 1,000,000,000
= 150,000,000,000 bytes
≈ 150 GB
```

So the theoretical storage requirement is approximately:

```text
150 GB / month
```

However, this is only a **back-of-the-envelope calculation**.

Real storage requirements will be higher because of:

- Database indexes
- Document/row overhead
- Replication
- Storage engine overhead
- Metadata
- Backups
- Fragmentation
- Deleted/expired URLs
- Operational overhead

Therefore, the actual storage requirement could be significantly higher.

---

# 4. Database Requirements

The system has a highly skewed workload:

```text
1 write : 100 reads
```

Therefore, the database layer should be optimized primarily for:

- Extremely high read throughput
- Low-latency lookups
- Horizontal scalability
- High availability
- Efficient key-based access

The primary lookup pattern is extremely simple:

```text
shortCode → originalURL
```

For example:

```text
aB7xK92p → https://example.com/very/long/url/...
```

This is essentially a **key-value lookup**, which makes NoSQL databases a strong candidate.

Possible choices include:

### MongoDB

MongoDB can store the URL mapping and metadata and provides:

- Horizontal scaling
- Indexing
- High availability
- Flexible schema
- Simple key-based lookups

### Cassandra

Cassandra is particularly suitable for extremely large-scale, distributed workloads with high availability and predictable key-based access patterns.

### Redis

Redis provides extremely fast in-memory lookups and is excellent as a **cache**.

However, using Redis as the sole permanent database may not be ideal for the entire URL dataset because the dataset can become very large and memory is considerably more expensive than persistent storage.

Therefore, Redis is better suited as a caching layer.

---

# 5. Proposed Database Architecture

A practical architecture would be:

```text
             Client
                │
                ▼
          Load Balancer
                │
                ▼
        URL Shortener API
           │           │
           │           │
           ▼           ▼
        Redis        MongoDB
        Cache       Persistent DB
```

For a redirect request:

```text
Client
  │
  │ tiny.url/aB7xK92p
  ▼
API Server
  │
  ▼
Redis
  │
  ├── Cache HIT ──────► Return original URL
  │
  └── Cache MISS
          │
          ▼
       MongoDB
          │
          ▼
      Original URL
          │
          ▼
      Store in Redis
          │
          ▼
       Redirect
```

This allows frequently accessed URLs to be served directly from Redis without repeatedly querying the persistent database.

Because the workload is approximately:

```text
1 write : 100 reads
```

caching can dramatically reduce database read pressure.

---

# 6. Database Schema

A basic URL document could look like:

```text
URL
--------------------------------
_id
shortCode
longUrl
createdAt
expiresAt
userId
clickCount
```

The most important field is:

```text
shortCode
```

which should have a unique index.

For example:

```text
shortCode: "aB7xK92p"

longUrl:
"https://example.com/products/very-long-url..."

createdAt:
2026-10-31

expiresAt:
2027-10-31
```

The primary lookup becomes:

```text
shortCode → longUrl
```

Therefore, we should create a unique index on `shortCode`.

---

# 7. Short-Code Generation

One of the most important parts of the system is generating the short code.

A naive approach would be to hash the original URL using algorithms such as:

```text
MD5
SHA-256
```

However, this is not ideal for our use case.

Hashing has two major problems.

### Problem 1 — Hashes are not reversible

We ultimately need to perform:

```text
shortCode → originalURL
```

A cryptographic hash does not allow us to recover the original URL.

We would therefore still need to store a mapping somewhere.

### Problem 2 — Collision handling

Different URLs can theoretically produce the same shortened hash/code.

For example:

```text
URL A → abc123
URL B → abc123
```

The probability depends on the hash space and implementation, but collision handling still adds complexity.

---

# 8. Base Conversion Approach

Instead of hashing the URL, we can generate a **unique numeric ID** for every URL and convert that ID into a compact representation using Base62 encoding.

Our character set would be:

```text
0123456789
abcdefghijklmnopqrstuvwxyz
ABCDEFGHIJKLMNOPQRSTUVWXYZ
```

Total:

```text
62 characters
```

Suppose a URL receives the unique ID:

```text
125789
```

We convert that number from base-10 into base-62.

The resulting value becomes something like:

```text
w7K
```

That becomes our short code:

```text
tiny.url/w7K
```

The important idea is:

```text
Unique ID
    ↓
Base62 encoding
    ↓
Short code
```

And during decoding:

```text
Short code
    ↓
Base62 decoding
    ↓
Unique ID
```

This is deterministic and does not require hashing.

---

# 9. Why Base62?

Base62 gives us a very large namespace while keeping the generated URLs short.

With 8 characters:

```text
62⁸
= 218,340,105,584,896
```

possible combinations.

This provides approximately:

```text
218 trillion
```

possible short codes.

Therefore, an 8-character Base62 namespace gives the system a very large capacity before we need to increase the code length.

---

# 10. Unique ID Generation

The remaining question is:

> How do we generate a globally unique ID?

Several approaches are possible:

### Database auto-increment ID

Simple, but difficult to scale across multiple database nodes.

### Distributed ID generator

A distributed ID generation system such as Snowflake-style IDs can generate unique IDs across multiple servers.

### Dedicated ID service

We can maintain a separate service responsible for generating unique IDs.

For a large-scale distributed system, a **distributed ID generator** is generally preferable.

The overall flow becomes:

```text
User submits long URL
        ↓
Generate unique ID
        ↓
Convert ID → Base62
        ↓
Store:
shortCode → longURL
        ↓
Return short URL
```

Example:

```text
Unique ID
   ↓
125789
   ↓
Base62
   ↓
w7K
   ↓
tiny.url/w7K
```

---

# 11. High-Level Architecture

A scalable architecture could look like:

```text
                    ┌──────────────┐
                    │    Client    │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │Load Balancer │
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
       ┌──────────────┐         ┌──────────────┐
       │ API Server 1 │         │ API Server 2 │
       └──────┬───────┘         └──────┬───────┘
              │                        │
              └────────────┬───────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │    Redis     │
                    │    Cache     │
                    └──────┬───────┘
                           │
                     Cache Miss
                           │
                           ▼
                    ┌──────────────┐
                    │   MongoDB    │
                    │  Persistent  │
                    │    Storage   │
                    └──────────────┘
```

The key design principle is:

> **Because URL redirects vastly outnumber URL creations, the system should optimize the read path aggressively using caching, horizontal scaling, and efficient key-based lookups.**

For the assumed workload, a practical starting point would be **MongoDB for persistent storage + Redis for caching**, with the architecture designed so that the database layer can later be replaced or scaled independently as traffic grows.