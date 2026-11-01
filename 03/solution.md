# URL Shortener: System Design (Detailed Notes)

> A deep-dive for interviews and real implementation. It covers requirements, capacity estimation, code generation, database choice, caching, expiry/purging, scaling, trade-offs, and the alternatives for each decision.

---

## Table of Contents

1. [Requirements](#1-requirements)
2. [Capacity Estimation](#2-capacity-estimation-back-of-the-envelope)
3. [Short Code Length and Keyspace](#3-short-code-length-and-keyspace)
4. [API Design](#4-api-design)
5. [Database Choice](#5-database-choice)
6. [Schema Design](#6-schema-design)
7. [Short Code Generation](#7-short-code-generation-the-core-problem)
8. [Read Path, Caching and Redirects](#8-read-path-caching-and-redirects)
9. [High-Level Architecture](#9-high-level-architecture)
10. [URL Expiry](#10-url-expiry)
11. [Database Purging](#11-database-purging)
12. [Scaling the System](#12-scaling-the-system)
13. [Analytics](#13-analytics-without-hurting-latency)
14. [Security and Abuse](#14-security-and-abuse-prevention)
15. [Failure Modes](#15-failure-modes-and-mitigations)
16. [Interview Talking Points](#16-interview-talking-points-cheat-sheet)

---

## 1. Requirements

### Functional
- Given a long URL, generate a unique short URL.
- Given a short URL, redirect to the original long URL.
- (Optional) Custom aliases, e.g. `tiny.url/my-brand`.
- (Optional) Expiry: links die after a configurable time.
- (Optional) Click analytics.

### Non-Functional
- **Highly available.** If the redirect service is down, every link on the internet that points to us breaks.
- **Low latency.** Redirects should take tens of milliseconds.
- **Read-heavy.** The read:write ratio is about 100:1.
- **Short codes must be unique** (no two long URLs share a code).
- **Codes should not be trivially guessable** (privacy).
- **Horizontally scalable** at every layer.

### Why define these first?
Every later decision (database, ID generation, caching) is a trade-off against these requirements. Stating them first shows the interviewer you design from constraints, not from favourite tools.

---

## 2. Capacity Estimation (Back-of-the-Envelope)

### Assumptions
| Item | Value |
|---|---|
| New URLs (writes) | 1 billion / month |
| Read:Write ratio | 100:1 |
| Reads | 100 billion / month |
| Avg long URL size | 100 bytes |
| Metadata per record | 50 bytes |

### Traffic
```
Seconds in a month  = 30 * 24 * 3600 = ~2.6 million

Write QPS (avg)     = 1,000,000,000 / 2,600,000   = ~385 writes/sec
Read QPS  (avg)     = 100,000,000,000 / 2,600,000 = ~38,500 reads/sec
Peak (assume 2-3x)  = ~100,000 reads/sec
```

### Storage
```
Per record  = 100 B (URL) + 50 B (metadata) = 150 B
Per month   = 150 B * 1 B writes            = 150 GB
Per year    = 150 GB * 12                   = 1.8 TB
5 years     = ~9 TB
```

> **Reality check.** This is the raw data size. Real storage will be roughly 2-3x higher because of indexes, replication (typically 3 copies), DB overhead, and fragmentation. Plan for around 25-30 TB over 5 years, replicas included.

### Bandwidth
```
Redirect response ~ 500 bytes (headers + Location)
Egress = 38,500 * 500 B = ~19 MB/s average
```
This is small. The bottleneck is **request rate and lookup latency**, not bandwidth.

### Cache Memory (80/20 rule)
About 20% of URLs generate about 80% of the traffic, so cache that 20%.
```
Reads per day      = 100 B / 30  = ~3.3 B/day
Cache 20% of that  = 0.2 * 3.3 B * 150 B = ~100 GB
```
~100 GB is easily spread across a small Redis cluster.

### Why do this at all?
It tells us the system is **read-dominated, small-record, and key-value shaped**. Those facts justify the NoSQL and cache-heavy architecture below.

---

## 3. Short Code Length and Keyspace

| Charset | Characters | 7 chars | 8 chars |
|---|---|---|---|
| Digits only (0-9) | 10 | 10^7 = 10 M | 10^8 = 100 M |
| Lowercase + digits | 36 | 36^7 = ~78 B | 36^8 = ~2.8 T |
| **Base62** (0-9, a-z, A-Z) | **62** | **62^7 = ~3.5 T** | **62^8 = 218,340,105,584,896 (~218 T)** |

### Analysis
- Digits only: 10^8 = 100 M is exhausted in about 3 months at 1 B writes/month. Not viable.
- Base62 with 7 chars: 3.5 T lasts 3.5 T / 12 B per year, about **290 years**.
- Base62 with 8 chars: 218 T lasts about **18,000 years**.

### Why 7 is usually enough, and when to pick 8
- 7 chars is the industry norm (shorter is the whole point of the product).
- 8 chars gives headroom if you use **random generation** (collision probability rises as the space fills; see section 7) or want sparse, hard-to-guess IDs.

### Why Base62 and not Base64?
Base64 adds `+` and `/` (and `=` padding), which are special characters in URLs and need escaping. The URL-safe variant swaps them for `-` and `_`. That is legal, but base62 is cleaner, copy-paste friendly, and double-click selectable.

### Drawbacks of Base62
- Visually ambiguous characters (`0/O`, `1/l/I`). Some systems drop them (Base58, as in Bitcoin) at the cost of a smaller keyspace.
- **Case-sensitive:** `abc` and `ABC` are different links. Users who type links by hand will make mistakes.
- Accidental profanity can appear in random codes. Production systems keep a blocklist.

---

## 4. API Design

```
POST /api/v1/urls
Body: { "longUrl": "https://example.com/very/long/path",
        "customAlias": "my-brand",      // optional
        "expiresAt": "2027-01-01T00:00Z" // optional
      }
Response 201: { "shortUrl": "https://tiny.url/aZ3x9Qk", "expiresAt": "..." }

GET /{shortCode}
Response 301/302: Location: <longUrl>
Response 404: not found
Response 410: gone (expired)

DELETE /api/v1/urls/{shortCode}   (authenticated)
```

### Notes
- Use **410 Gone** for expired links and **404** for never-existed ones. It is semantically correct and helps debugging.
- Validate `longUrl` (scheme must be http/https, max length, not our own domain) to prevent open-redirect loops and abuse.
- Make `POST` **idempotent** where possible (same user + same long URL returns the same short URL) if you want dedup. It costs an extra index lookup, so decide whether dedup is worth it (see section 7).

---

## 5. Database Choice

### Access pattern analysis
- Lookup is **always by key** (`shortCode` to `longUrl`).
- **No joins, no multi-row transactions**, no complex queries.
- Huge volume, tiny records, read-heavy, with eventual consistency being acceptable for reads.

That is a textbook **key-value / document** workload.

### Options

| Database | Role | Pros | Cons |
|---|---|---|---|
| **MongoDB** | Primary store | Easy horizontal sharding, native TTL indexes, flexible schema, great with Node/MERN | Weaker write throughput than Cassandra at extreme scale, needs careful shard-key choice, memory-hungry working set |
| **Cassandra** | Primary store | Massive write scalability, masterless (no SPOF), tunable consistency, native per-row TTL | Operationally heavy, query flexibility is poor (design tables per query), tombstone issues with heavy deletes/TTL |
| **DynamoDB** | Primary store (managed) | Zero-ops, auto-scaling, native TTL | Vendor lock-in, cost at high read volume (mitigated by cache), hot-partition risk |
| **Redis** | **Cache layer** | Sub-millisecond reads, native key TTL | RAM is expensive, so 150 GB+/month of data is costly; persistence (RDB/AOF) is weaker than a real DB; not ideal as the sole source of truth |
| **PostgreSQL/MySQL** | Alternative primary | Strong consistency, familiar, ACID, easy unique constraints | Sharding is manual (Vitess/Citus), relational features unused, harder to scale writes horizontally |

### Recommendation: **MongoDB (primary) + Redis (cache)**

**Why:**
1. 150 GB/month growth means we must **shard**. Mongo has built-in sharding.
2. Native **TTL index** gives us free expiry.
3. Redis in front absorbs the ~80% of reads that hit hot links, so Mongo sees only a fraction of the 38K+ QPS.
4. It fits the user's existing MERN skill set, so there is lower operational and learning cost.

### Why not SQL?
SQL *can* work. Many large shorteners ran on sharded MySQL. The honest answer is: **we don't need what SQL is good at** (joins, transactions), and we *do* need what NoSQL is good at (cheap horizontal scaling). A strong rebuttal is that a unique constraint on `shortCode` is trivially provided by Mongo's `_id` too.

### Why not Redis alone?
- Cost: 150 GB/month in RAM, replicated, is far more expensive than disk.
- Durability: a Redis crash and restore from an old snapshot means **permanently broken short links**, which is a product-killing failure.
- Use it for what it is best at: a **cache** with TTL.

### Consistency consideration
URL mappings are **write-once, read-many** and rarely updated. That makes caching easy (almost no invalidation problems) and makes **eventual consistency** acceptable on replicas. The one exception is the window right after creation. Fix it with read-your-writes (read from the primary, or write-through to cache on creation).

---

## 6. Schema Design

### MongoDB document

```js
// models/Url.js (Mongoose)
const mongoose = require('mongoose');

const urlSchema = new mongoose.Schema(
  {
    _id: { type: String, required: true },     // the shortCode itself
    longUrl: { type: String, required: true, maxlength: 2048 },
    userId: { type: String, index: true, default: null },
    createdAt: { type: Date, default: Date.now },
    expiresAt: { type: Date, default: null },  // null = never expires
    isCustomAlias: { type: Boolean, default: false },
  },
  { versionKey: false }
);

// TTL index: Mongo's background thread deletes docs when expiresAt < now
urlSchema.index({ expiresAt: 1 }, { expireAfterSeconds: 0 });

module.exports = mongoose.model('Url', urlSchema);
```

### Design decisions and the whys

| Decision | Why | Drawback |
|---|---|---|
| `shortCode` as `_id` | `_id` already has a unique index, so no extra index is needed and lookups are one B-tree hit | Immutable; you can't rename a code without delete + insert |
| Shard key = hashed `_id` | Even distribution; no hot shard from sequential IDs | Range queries (e.g. "all links created today") become scatter-gather |
| `expiresAt` with `expireAfterSeconds: 0` | Expiry time is per-document, with the exact date set in the field | TTL deletion is **not instantaneous** (see section 10) |
| Keep analytics out of this doc | Hot-path read stays tiny; write amplification avoided | Needs a separate analytics store |

### Optional secondary index
- `{ userId: 1, createdAt: -1 }` if users need a "my links" dashboard.
- `{ longUrl: 1 }` only if you want dedup. It is a **big** index (URLs are long), so consider indexing a hash of the URL instead.

---

## 7. Short Code Generation (The Core Problem)

### 7.1 Why not just use SHA-256 / MD5?

A common (but slightly wrong) explanation is "hashes are not reversible". In reality **we never need to reverse the code**. We store `code -> longUrl` and look it up. The real problems are:

1. **Length:** MD5 gives 128 bits (32 hex chars); SHA-256 gives 256 bits. We need 7 characters, so we must **truncate**.
2. **Truncation causes collisions.** Taking the first 7 base62 chars means you only keep about 41 bits (62^7 ~ 2^41.7). By the **birthday paradox**, collisions become likely after about sqrt(3.5 T) ~ **1.9 million** URLs. We're creating 1 billion a month, so collisions would be constant.
3. **Same input gives the same output.** Two users shortening the same URL get the same code, which prevents per-user analytics or expiry unless you salt the input.
4. **Collision handling costs a DB read before every write**, and retries add latency.

### 7.2 Approaches compared

#### Approach A: Hash + truncate + collision check
```
code = base62(md5(longUrl + salt)).slice(0, 7)
if exists(code): salt++ and retry
```
- **Pros:** Stateless, with no central coordinator. Same URL gives the same code (free dedup if you want it).
- **Cons:** Collisions become frequent at scale, so every write needs an existence check (read before write). Retry loops under load. Hot DB reads on the write path.
- **Verdict:** Fine for a small system; poor at 1 B/month.

#### Approach B: Random code + collision check
```
code = randomBase62(7)
insert; if duplicate key error -> retry
```
- **Pros:** Simple. Codes are **unpredictable**, which is good for privacy.
- **Cons:** Collision probability grows as the keyspace fills. At 1 T of 3.5 T used (~28%), nearly 1 in 3.5 inserts collides. Each retry costs a round-trip. Relies on the DB's unique constraint as the arbiter.
- **Verdict:** Acceptable with 8+ chars (the space stays sparse for centuries). Cheap in code, but still wastes DB calls.

#### Approach C: Auto-increment counter + Base62 (the one in your notes)
```
id   = nextId()         // 1, 2, 3, ...
code = base62(id)
```
- **Pros:** **Zero collisions by construction.** No read-before-write. Shortest possible codes (they grow as the counter grows). Very fast.
- **Cons:**
  - **Predictable and enumerable:** a competitor can scrape `.../1`, `.../2`, `.../3` and learn your entire link volume and content.
  - A **single counter is a SPOF and a bottleneck** in a distributed system.
  - Needs coordination across servers.
- **Verdict:** The best base, if you solve the distributed counter problem and the predictability problem (both below).

#### Approach D: Key Generation Service (KGS), pre-generated keys
A separate service **pre-generates** millions of unique random codes and stores them in two tables, `unused_keys` and `used_keys`. App servers fetch **batches** (e.g. 1,000 keys) into memory and hand them out locally.
- **Pros:** No collisions, no computation on the write path, unpredictable codes, very fast.
- **Cons:** Extra service to operate. The KGS is a SPOF unless replicated. If an app server crashes holding a batch, those keys are **lost** (acceptable: 3.5 T keys is plenty). Needs locking so two servers never get the same batch.
- **Verdict:** Excellent and widely used. More infrastructure.

#### Approach E: Snowflake-style distributed IDs
64-bit ID = `timestamp (41 bits) | machine ID (10 bits) | sequence (12 bits)`, then base62-encoded.
- **Pros:** No central coordinator. Roughly time-ordered. Each machine generates 4096 IDs/ms independently.
- **Cons:** 64-bit base62 gives up to **11 characters**, which is too long for our 7-char target. Requires clock sync (NTP drift can cause duplicates or rejects). Machine IDs must be assigned uniquely.
- **Verdict:** Great for general IDs; poor fit when code length is a core product feature.

### 7.3 Recommended: Counter ranges + Base62 + scrambling

**Distributed counter with range allocation.** A central store (ZooKeeper / etcd / a Mongo `counters` collection) hands out **ranges** of numbers. Each app server keeps its range in memory and increments locally with no network call.

```
App Server 1 -> gets range 1,000,000 - 1,000,999
App Server 2 -> gets range 1,001,000 - 1,001,999
...
When a server exhausts its range, it asks for the next one.
```

- **Pros:** The central store is hit once per 1,000 writes (not every write). It can be down for a short while without stopping writes, as servers use their remaining range.
- **Cons:** A crash loses the unused part of a range, so there are gaps. That is harmless given the keyspace size. The central store still needs its own HA setup.

### 7.4 Base conversion: how it works

A number is just a representation in some base. We pick base 62 with a custom alphabet:

```
Alphabet: 0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ
Index:    0 ......... 9 10 ........ 35 36 ................... 61
```

**Algorithm (encode):** repeatedly divide by 62, collecting remainders; the remainders (reversed) map to characters.

```
125 in base 62:
125 / 62 = 2 remainder 1   -> '1'
  2 / 62 = 0 remainder 2   -> '2'
Read remainders bottom-up -> "21"
```

**Implementation (use BigInt, since IDs can exceed 2^53):**

```js
const ALPHABET = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
const BASE = BigInt(ALPHABET.length); // 62n

function encodeBase62(num) {
  let n = BigInt(num);
  if (n === 0n) return ALPHABET[0];
  let out = '';
  while (n > 0n) {
    out = ALPHABET[Number(n % BASE)] + out;
    n = n / BASE;
  }
  return out;
}

function decodeBase62(str) {
  let n = 0n;
  for (const ch of str) {
    const idx = ALPHABET.indexOf(ch);
    if (idx === -1) throw new Error('Invalid character');
    n = n * BASE + BigInt(idx);
  }
  return n;
}

// encodeBase62(125)   -> "21"
// decodeBase62("21")  -> 125n
```

### 7.5 Why base conversion beats hashing here
- **Deterministic and collision-free:** distinct integers always map to distinct strings (it is a bijection).
- **Compact:** a bigger base needs fewer characters (base 10 needs about 11 digits for 62^6; base 62 needs 6 chars).
- **Reversible:** you can decode back to the ID. This is not required for lookup, but is useful for debugging, sharding by ID range, and analytics.

### 7.6 Fixing predictability (the main drawback of counters)

Sequential codes leak volume and allow enumeration. Options:

1. **Bijective scrambling:** map the counter through a reversible function before encoding, such as multiplying by a large odd constant modulo 62^7, or a small Feistel network. The mapping stays one-to-one, so there are no collisions, but codes look random.
   ```js
   const SPACE = 62n ** 7n;
   const MULT  = 2654435761n;  // must be coprime with SPACE (odd, not divisible by 31)
   const scramble = (id) => (BigInt(id) * MULT) % SPACE;
   const code = encodeBase62(scramble(counter)).padStart(7, '0');
   ```
2. **Use a library:** Sqids / Hashids encode integers into non-sequential short strings.
3. **Go to KGS with random keys** (Approach D).

> **Caveat:** Scrambling is *obfuscation*, not security. Do not rely on it for private links. If a link must be secret, use a long random token and authentication.

### 7.7 Custom aliases
- Check availability with a unique-insert (let the DB reject duplicates).
- **Namespace conflict:** a user may pick `abc1234` that the counter would generate later. Fix: reserve a different length or prefix for custom aliases (e.g. aliases must be 4-6 chars or contain `-`), or check on every generated insert and retry.
- Maintain a **reserved words list** (`api`, `admin`, `login`, `static`).

### 7.8 Should identical long URLs reuse the same code?
| Dedup | Pros | Cons |
|---|---|---|
| Yes | Saves storage; same input gives same output | Extra index on `longUrl`, extra read per write, and per-user analytics/expiry become tangled |
| No | Fast writes, per-link independent settings, simple | Duplicates consume storage (small at 150 B each) |

Most real systems choose **no dedup** for simplicity and independent link settings.

---

## 8. Read Path, Caching and Redirects

### Flow (cache-aside pattern)
```
Client -> CDN/LB -> App Server
                      |
                      +--> Redis GET code
                      |      |-- HIT  -> redirect
                      |      +-- MISS -> Mongo findById
                      |                    |-- found -> SET Redis (TTL) -> redirect
                      |                    +-- not found -> 404 (cache negative result briefly)
```

### Implementation sketch

```js
const redis = require('./redis');   // ioredis client
const Url   = require('./models/Url');

app.get('/:code', async (req, res) => {
  const { code } = req.params;

  // 1. cache
  const cached = await redis.get(`u:${code}`);
  if (cached) {
    if (cached === '__NF__') return res.sendStatus(404);
    return res.redirect(302, cached);
  }

  // 2. database
  const doc = await Url.findById(code).lean();
  if (!doc) {
    await redis.set(`u:${code}`, '__NF__', 'EX', 60);   // negative caching
    return res.sendStatus(404);
  }

  // 3. lazy expiry check (don't trust the TTL monitor alone)
  if (doc.expiresAt && doc.expiresAt <= new Date()) {
    return res.sendStatus(410);
  }

  // 4. populate cache; TTL = min(24h, time until expiry)
  const ttl = doc.expiresAt
    ? Math.min(86400, Math.floor((doc.expiresAt - Date.now()) / 1000))
    : 86400;
  if (ttl > 0) await redis.set(`u:${code}`, doc.longUrl, 'EX', ttl);

  return res.redirect(302, doc.longUrl);
});
```

### Cache details and the whys

- **Cache-aside** (lazy loading) is chosen because only requested links get cached, and a cache outage doesn't break correctness (falls back to DB).
- **Alternative: write-through** (cache on creation). It avoids the first-read miss, but caches links that may never be clicked.
- **Eviction policy:** `allkeys-lru` (or `allkeys-lfu`, better for skewed traffic) to keep hot links.
- **Negative caching:** caches "not found" briefly, which protects the DB from scanners hammering random codes.
- **Cache TTL must not outlive link expiry**, or expired links keep redirecting.

### Cache problems to mention
| Problem | What happens | Mitigation |
|---|---|---|
| **Cache stampede** | A hot key expires and thousands of requests hit the DB simultaneously | Request coalescing / per-key mutex, TTL jitter, stale-while-revalidate |
| **Hot key** | One viral link saturates a single Redis shard | Local in-process LRU cache (a few seconds), key replication, CDN caching |
| **Cache penetration** | Attackers request non-existent codes (always a miss) | Negative caching, **Bloom filter** of valid codes, rate limiting |
| **Stale data after delete** | Deleted link still redirects from cache | Explicit `DEL` on delete; short TTL |

### 301 vs 302: a classic interview question
| | 301 (Permanent) | 302 (Temporary) |
|---|---|---|
| Browser caching | Browser caches the redirect; **future clicks skip our server** | Browser asks us every time |
| Server load | Lower | Higher |
| Analytics | **Lost** after the first click per browser | Every click is counted |
| Changing/expiring the target | Hard (browser remembers) | Easy |
| SEO | Passes link equity to the target | Doesn't |

**Choice:** Use **302** if you need analytics, expiry, or the ability to edit/delete links. Use **301** only if you want to minimize load and never need to revoke links. With expiry as a stated requirement, **302 is the right default**.

---

## 9. High-Level Architecture

```
                         +-----------+
   Users --> DNS --> CDN |   (edge)  |
                         +-----+-----+
                               |
                       +-------v--------+
                       |  Load Balancer |  (L7, health checks, TLS termination)
                       +---+--------+---+
                           |        |
              +------------v+      +v------------+
              | App Server 1|      | App Server N|   (stateless)
              +--+-------+--+      +--+-------+--+
                 |       |            |       |
          +------v--+  +-v-------+  ...       |
          |  Redis  |  | ID/Range|            |
          | Cluster |  | Service |            |
          +------+--+  +---------+            |
                 | miss                       |
          +------v----------------------------v------+
          |   MongoDB Sharded Cluster (replica sets)  |
          +---------------------+---------------------+
                                |
              +-----------------+------------------+
              |                                    |
      +-------v-------+                   +--------v--------+
      | Expiry/Purge  |                   | Analytics Queue |
      | Workers (cron/|                   | (Kafka/BullMQ)  |
      | BullMQ)       |                   +--------+--------+
      +---------------+                            |
                                          +--------v--------+
                                          | Analytics Store |
                                          +-----------------+
```

### Load balancer: why and what to say
- **Why:** At about 100K peak QPS, no single server can cope; the LB spreads traffic and removes unhealthy nodes.
- **Algorithms:** round-robin (simple), least-connections (better for uneven request time), consistent hashing (useful if you want local caches per server to be effective).
- **Drawbacks:** The LB itself can be a SPOF, so run it in an active-passive/active-active pair (or a managed LB). Use **stateless** app servers so any server can serve any request.
- **Separate read and write services.** Reads are ~100x more frequent, so scale them independently (e.g. 50 read pods and 2 write pods). This is a cheap and effective optimization.

---

## 10. URL Expiry

### Requirement
Links may have a lifetime. After that they must stop redirecting, and their data should eventually be removed to save storage.

Two separate concerns:
1. **Logical expiry:** the link stops working *immediately* at expiry time.
2. **Physical deletion:** the data is reclaimed *eventually*.

### Approach 1: Database-native TTL (recommended primary mechanism)

**MongoDB TTL index**
```js
db.urls.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 })
```
- A background thread runs about **every 60 seconds** and removes documents where `expiresAt < now`.
- **Pros:** No extra code or infrastructure; atomic with the DB; battle-tested.
- **Cons:**
  - **Not precise:** deletion can lag by a minute or much more under heavy load, so you **must also check `expiresAt` on read** (lazy expiry).
  - The deletion load runs on the primary and can cause IO spikes with huge batches.
  - Only works on a single date field; cannot express complex retention rules (e.g. "delete only if no clicks in 90 days").
  - Can't hook custom logic (archive before delete, notify, update analytics).

**Redis key TTL**
```js
await redis.set(`u:${code}`, longUrl, 'EX', secondsUntilExpiry);
```
- Redis uses **lazy expiry** (on access) plus **periodic sampling** (about 20 keys per cycle, 10 times/sec), so memory is reclaimed approximately but not instantly. Expired keys never return stale data.
- **Cons:** Applies only to the cache. It does **not** delete the source-of-truth record in Mongo.

### Approach 2: Scheduled cron job
```js
// run every hour: delete in small batches
const cron = require('node-cron');
cron.schedule('0 * * * *', async () => {
  let deleted;
  do {
    const batch = await Url.find({ expiresAt: { $lte: new Date() } })
                           .select('_id').limit(1000).lean();
    if (!batch.length) break;
    const ids = batch.map(d => d._id);
    await Url.deleteMany({ _id: { $in: ids } });
    await redis.del(ids.map(id => `u:${id}`));
    deleted = ids.length;
  } while (deleted === 1000);
});
```
- **Pros:** Simple. Full control (batching, archiving, off-peak scheduling, metrics). Same idea used by trading platforms for scheduled jobs such as daily market updates.
- **Cons:**
  - If you run it on **multiple instances, every instance fires the job**, causing duplicate work and races. You need a **distributed lock** or leader election.
  - A big backlog with a full-collection scan is expensive; the `expiresAt` index is mandatory.
  - If the job crashes, nothing retries it unless you add monitoring.

### Approach 3: Job queue (BullMQ / Redis-backed)
- A **repeatable job** schedules the sweep; workers process delete batches in parallel.
- Or schedule a **delayed job per link** at creation time, firing at `expiresAt`.
- **Pros:** Retries, backoff, concurrency control, observability dashboards, and a single scheduler even with many instances (the queue guarantees one worker takes each job).
- **Cons:**
  - Extra infrastructure (Redis dependency for BullMQ).
  - **Per-link delayed jobs don't scale:** 1 B jobs/month would flood Redis. Use **sweeper jobs** that process *batches* instead.
  - If Redis loses queue data, scheduled expiries are lost (a sweeper fallback is wise).

### Approach 4: Lazy expiry (check on read)
Check `expiresAt` on every redirect and return 410 if expired.
- **Pros:** Exact, instant logical expiry with no background process.
- **Cons:** Doesn't free storage on its own. Dead links that are never visited stay forever.
- **Verdict:** Always use it **together with** a physical deletion mechanism.

### Approach 5: Time-partitioned tables/collections
Partition data by expiry month. Expiring a month means **dropping the whole partition**, which is nearly free (no row-by-row deletes).
- **Pros:** Extremely efficient bulk purge, no tombstones, no index churn.
- **Cons:** Requires fixed expiry granularity (e.g. expiry rounded up to month-end), more complex routing and queries.

### Recommended combination
> **Lazy check on read** (correctness) + **Mongo TTL index** (cheap physical cleanup) + **Redis TTL capped at link expiry** (cache correctness) + **BullMQ/cron sweeper** for complex rules (archiving, inactivity-based purge, reconciliation).

---

## 11. Database Purging

**Database purging** is the process of permanently removing obsolete, unwanted, or temporary data from a database to manage storage, improve query performance, and comply with data retention policies.

### Why purge?
- **Storage cost:** 150 GB/month adds up to petabyte-scale replicas over time.
- **Performance:** Smaller indexes fit in RAM; fewer pages to scan; faster backups and restores.
- **Compliance:** GDPR "right to erasure", data-retention laws, user account deletion.
- **Hygiene:** Abandoned, spam, or malicious links should not live forever.

### What to purge
| Data | Trigger |
|---|---|
| Expired links | `expiresAt < now` |
| Inactive links | No clicks in N months (policy decision, e.g. free tier) |
| Spam / malicious links | Abuse detection or reports |
| Deleted-user data | Account deletion / privacy request |
| Old analytics detail | Roll up to daily aggregates, then delete raw events |

### Purging strategies and trade-offs

**Hard delete**
- Permanently removes data. **Pros:** reclaims space, satisfies erasure requests. **Cons:** irreversible; mistakes are expensive.

**Soft delete (`isDeleted` flag)**
- Mark first, hard-delete later. **Pros:** undo window, audit trail. **Cons:** the data still costs storage, and every query must filter on the flag.

**Archive then purge**
- Copy to cold storage (S3 / Glacier) and then delete. **Pros:** cheap long-term retention, supports audits. **Cons:** extra pipeline, retrieval latency.

### Practical pitfalls
- **Delete in small batches** (500-1000) with pauses, so you don't lock collections or spike replication lag.
- **Run during off-peak hours** and monitor replication lag.
- **Cassandra:** deletes and TTL create **tombstones**, which hurt read performance until compaction. Use time-window compaction, or time-bucketed tables.
- **MongoDB:** deleted space is reused by the engine but often **not returned to the OS** until compaction or `compact`/resync. Disk usage may not fall right away.
- **Always purge the cache and secondary stores as well** (Redis, search index, analytics), or you get "ghost" links.

### Should expired short codes be recycled?
- **Pro:** Reuses keyspace.
- **Con:** Old links posted on the web, in emails, or printed material would suddenly point to **someone else's content**. This is a **security and trust issue**.
- With 3.5 T+ keys, there is **no need to recycle**. Treat codes as permanent identifiers (return 410 forever).

---

## 12. Scaling the System

### Database sharding
| Strategy | How | Pros | Cons |
|---|---|---|---|
| **Hash-based** on `shortCode` | `hash(code) % N` / hashed shard key | Even load, no hot shards | Range queries scatter; resharding is expensive without consistent hashing |
| **Range-based** on code | Shard by first character/ID range | Range queries efficient | **Hot shards** (new IDs always hit the newest range) |
| **Consistent hashing** | Hash ring with virtual nodes | Adding/removing nodes moves only ~1/N of keys | More complex to implement and operate |

**Recommendation:** hashed shard key on `_id`.

### Replication
- Replica sets (1 primary + 2 secondaries), 3 copies of every document.
- Reads can go to secondaries (**eventual consistency** is fine for immutable mappings).
- **Drawback:** replication lag means a just-created link may 404 on a lagging replica. Fix: write-through cache on creation, or read-your-writes from the primary for a few seconds.

### Multi-region
- Replicate the data and serve redirects from the nearest region (GeoDNS / anycast).
- **Drawback:** cross-region write conflicts and higher cost. Since writes are only about 385/sec, one write region plus many read regions is typically enough.

### CDN
- Cache redirects at the edge for very hot links.
- **Drawback:** only works well with 301 or with short `Cache-Control` on 302; analytics and instant revocation are affected.

### Database connection pools
Use pooled connections and limits per app server so a traffic spike doesn't exhaust DB connections.

---

## 13. Analytics Without Hurting Latency

Never write analytics to the DB synchronously inside the redirect handler.

```
Redirect handler -> push event {code, ts, ip, ua, referrer} to queue (fire-and-forget)
                    -> consumers batch-write to an analytics store
```

- **Queue options:** Kafka (high throughput, replay), BullMQ/Redis Streams (simpler), SQS (managed).
- **Store options:** ClickHouse / Druid (OLAP, fast aggregates), Cassandra (time-series), or Mongo time-series collections for smaller scale.
- **Why separate stores:** the analytics workload is append-heavy and aggregate-heavy; mixing it with the lookup DB would slow the hot path.
- **Drawbacks:** counts are **eventually consistent**, a queue outage can drop events (accept loss or use durable queues), and click counts have bot/duplicate noise.

---

## 14. Security and Abuse Prevention

- **Rate limiting:** per IP and per API key (token bucket in Redis) to stop mass link creation and enumeration.
- **URL validation:** allow only `http/https`; block `javascript:`, `data:`, `file:`; block internal IPs (SSRF protection if you fetch titles/previews).
- **Malicious URL detection:** check against Google Safe Browsing or similar; scan on creation and periodically (links can turn malicious later).
- **Authentication** for creation/deletion APIs.
- **Unpredictable codes** to limit scraping (scrambling or random codes).
- **Reserved words** blocklist for custom aliases, and a profanity filter for generated codes.
- **HTTPS everywhere**, HSTS.
- **Interstitial warning page** for flagged links.

---

## 15. Failure Modes and Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Redis down | All reads hit Mongo; potential overload | Redis cluster with replicas/sentinel; DB capacity headroom; local in-memory fallback cache; rate limiting |
| ID/range service down | Writes stall once local ranges are exhausted | Large ranges per server, replicated coordinator (etcd/ZooKeeper), range pre-fetch |
| App server crash with unused range | Gaps in the ID space | Acceptable given the keyspace size |
| Mongo primary failover | Brief write unavailability | Replica set auto-election, retry with backoff |
| Replication lag | Newly created link 404s | Write-through cache; read primary briefly |
| Clock skew (if using timestamp IDs) | Duplicate or out-of-order IDs | NTP, monotonic checks, or avoid time-based IDs |
| Expiry job double-run | Duplicate deletes (harmless if idempotent) | Distributed lock; idempotent deletes |
| Viral link | Hot key | Local cache + CDN + Redis replicas |

---

## 16. Interview Talking Points (Cheat Sheet)

### Order of discussion
1. Clarify requirements, scale, and read:write ratio.
2. Estimate capacity (QPS, storage, cache).
3. Choose code length and charset (justify 7 vs 8, base62).
4. Define API and data model.
5. Pick the database with reasons (and say why not the alternatives).
6. **Code generation deep dive** (the heart of the question).
7. Caching and redirect semantics (301 vs 302).
8. Expiry, then purging.
9. Scaling: sharding, replication, LB, CDN.
10. Analytics, security, failure handling.

### One-line justifications
- **Why base62?** Largest URL-safe alphanumeric set, with no escaping, giving 62^7 ~ 3.5 T codes in 7 chars.
- **Why not hash?** Truncation to 7 chars makes collisions inevitable by the birthday bound; it also forces a read-before-write.
- **Why counter + base conversion?** It is a bijection, so there are zero collisions, and a range-allocated counter avoids per-write coordination.
- **Why NoSQL?** Pure key-value access, huge volume, need for easy horizontal scale; no joins or transactions required.
- **Why Redis as cache, not DB?** RAM cost and durability; cache-aside keeps the DB as the source of truth.
- **Why 302?** Keeps analytics and revocation possible, at the cost of more server load.
- **Why TTL index plus lazy check?** TTL deletion is approximate, so lazy expiry gives exactness and the TTL index gives cleanup.
- **Why not recycle codes?** Old links could redirect to unrelated, possibly malicious content.

### Trade-off summary table

| Decision | Chosen | Main alternative | Why chosen | Main drawback of chosen |
|---|---|---|---|---|
| Code generation | Range counter + base62 + scramble | Hash+truncate / KGS / Snowflake | No collisions, short codes, cheap writes | Needs a coordinator; scrambling is only obfuscation |
| Primary DB | MongoDB (sharded) | Cassandra / DynamoDB / SQL | Native TTL and sharding, fits MERN | Shard key must be right; RAM-hungry working set |
| Cache | Redis, cache-aside | Memcached / write-through | Rich features, TTL, clustering | Stampede, hot keys, extra moving part |
| Expiry | Lazy + TTL index + sweeper | Cron only / BullMQ per-link jobs | Exact and cheap, with a fallback | Several mechanisms to reason about |
| Redirect | 302 | 301 | Analytics, revocation | Higher server load |
| Analytics | Async via queue | Sync DB write | Protects redirect latency | Eventual consistency, possible event loss |

### Questions to expect and short answers
- **"What if two servers generate the same ID?"** They can't: the ranges are disjoint, and the unique `_id` is the final safety net.
- **"What if the DB runs out of the keyspace?"** Move from 7 to 8 chars. Existing codes stay valid because all lengths coexist (shorter codes are just a prefix-free subset by length).
- **"How do you handle a celebrity link at 1M QPS?"** CDN plus a local in-process cache plus Redis read replicas.
- **"How do you delete a link everywhere?"** Delete in the DB, `DEL` the Redis key, purge the CDN, and rely on the 302 so browsers don't cache it.
- **"How would you support link editing?"** Use 302, update the DB, and invalidate the cache. This is impossible with 301 once browsers cache it.

---

---

## 17. Conclusion

### The problem in one paragraph
A URL shortener looks trivial, but at **1 billion writes and 100 billion reads per month** it becomes a system design problem about three things: **generating unique short codes without collisions, serving about 38K reads/sec (100K at peak) with very low latency, and managing 150 GB+ of new data every month** (about 1.8 TB per year, before replication and indexes).

### Final design at a glance

| Concern | Final decision | Core reason |
|---|---|---|
| Code length and charset | 7 characters, Base62 | 62^7 ≈ 3.5 T codes, about 290 years of capacity, URL-safe with no escaping |
| Code generation | Range-allocated counter + Base62 + bijective scrambling | Zero collisions by construction, no read-before-write, unpredictable-looking codes |
| Primary database | MongoDB, sharded on hashed `_id`, replica sets | Key-value access pattern, easy horizontal scaling, native TTL index |
| Cache | Redis, cache-aside, LRU/LFU eviction | About 80% of traffic hits about 20% of links; absorbs the read load |
| Redirect type | HTTP 302 | Keeps analytics, expiry, and link revocation possible |
| Expiry | Lazy check on read + Mongo TTL index + Redis TTL capped at link expiry | Lazy check gives exactness; TTL index gives cheap cleanup |
| Purging | Batched hard delete (optionally soft delete or archive first) via cron/BullMQ sweeper | Reclaims storage, keeps indexes small, meets retention and compliance needs |
| Scaling | Stateless app servers behind a load balancer, separate read/write services, CDN for hot links | Reads are about 100x writes, so scale them independently |
| Analytics | Async via queue (Kafka/BullMQ), separate store | Never slows the redirect path |

### Key lessons

1. **Read the access pattern first.** Lookup is always `code -> longUrl`, with no joins and no transactions. That single fact drives the choice of NoSQL, caching, and sharding.
2. **Hashing is the wrong tool for short codes.** The issue is not that hashes are irreversible; it is that truncating a hash to 7 characters makes collisions inevitable (birthday bound at about 1.9 M URLs). A counter plus base conversion is a bijection, so collisions are impossible.
3. **Predictability is the hidden cost of counters.** Fix it with scrambling, Sqids/Hashids, or a Key Generation Service. Remember that this is obfuscation, not security.
4. **Redis is a cache, not the source of truth.** RAM is expensive and a lost snapshot means permanently broken links.
5. **Expiry has two halves.** Logical expiry (the link stops working now) and physical deletion (storage is reclaimed later) need different mechanisms. Use both.
6. **Never recycle expired codes.** With trillions of codes available, reuse only creates risk: old links could redirect to someone else's content.
7. **Protect the hot path.** Analytics, purging, and abuse scanning all run asynchronously, away from the redirect request.

### Alternatives and when they win

| If your situation is... | Consider instead |
|---|---|
| Small scale (under 10 M links), team already knows SQL | A single PostgreSQL/MySQL instance with a unique index; skip sharding |
| Extreme write throughput, multi-region active-active | Cassandra or DynamoDB as the primary store |
| Zero-ops preference, AWS-native | DynamoDB with TTL + ElastiCache |
| Need unpredictable codes with no scrambling logic | Key Generation Service with pre-generated random keys |
| Need a fully decentralized ID generator | Snowflake-style IDs (accepting longer codes) |
| Want to minimize server load and never revoke links | 301 redirects (losing analytics and editability) |
| Millions of per-link timed events needed | Time-partitioned collections and drop whole partitions |

### Known limitations of the chosen design
- **Operational complexity:** Mongo shards, Redis cluster, a range-allocation service, a queue, and sweeper jobs. Each is a thing to monitor and fail over.
- **Eventual consistency:** replica lag can briefly 404 a new link, and analytics counts lag behind real clicks.
- **Coordinator dependency:** the range service must be highly available, though large ranges give a buffer.
- **TTL imprecision:** physical deletion lags expiry, so correctness depends on the lazy check.
- **Capacity numbers are estimates:** real storage will be about 2-3x raw size once indexes, replication, and overhead are included.

### Future improvements
- Bloom filter in front of the cache to block non-existent-code lookups.
- Tiered storage: move cold, rarely clicked links to cheaper storage.
- Geo-distributed read replicas and edge redirects for global latency.
- Machine-learning-based malicious URL detection, with periodic re-scanning of existing links.
- Link-level features: password protection, click limits, QR codes, and per-link custom domains.

### Final takeaway
> The right URL shortener design is not about picking the trendiest database. It is about matching each component to the workload: **a collision-free, range-allocated counter with base62 encoding for codes, a sharded document store for durability, Redis for read speed, 302 redirects for control, and layered expiry and purging for hygiene.** Every choice trades something away, and a strong answer states the choice, the reason, the alternative, and the cost.


*End of document.*