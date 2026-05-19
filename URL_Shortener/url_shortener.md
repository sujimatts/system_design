# Designing a URL Shortener

Turn a long URL into something like `short.ly/3d7`, and redirect back to the original when clicked.

---

## Core Idea

> You don't shorten the URL. You assign it a tiny ID and store the mapping in a database.

A URL shortener is just a **lookup table** with a clever ID scheme.

---

## Scale Estimate

- 100M new URLs/day → ~1,200 writes/sec
- Reads are ~100× writes → ~120,000 reads/sec
- **Read-heavy** → caching matters
- 5 years × 500 bytes/row → ~90 TB

---

## Generating the Short Code

**Counter + Base62** (recommended over hashing — no collisions).

1. Give each URL a unique number from a counter (1, 2, 3, …).
2. Convert that number to Base62 → short URL-safe string.

### The Counter

- 1st URL → `1`, 2nd → `2`, …
- Use **Redis `INCR`** — atomic, fast. Alternatives: range allocation per server, or Snowflake IDs.

### Base62

Uses 62 URL-safe symbols: `0-9`, `a-z`, `A-Z`.

**Convert 12345 to Base62:**

```
12345 ÷ 62 = 199  remainder  7   →  "7"
  199 ÷ 62 =   3  remainder 13   →  "d"
    3 ÷ 62 =   0  remainder  3   →  "3"
```

Read bottom-up → **`3d7`**

| Chars | Base10        | Base62           |
|-------|---------------|------------------|
| 6     | 1,000,000     | 56 billion       |
| 7     | 10,000,000    | **3.5 trillion** |

7 chars covers internet scale for centuries.

---

## Walkthrough — Shortening `www.google.com/api/paste/new/cricket`

1. `INCR` counter → `12345`   #INCR is a redis commands. It means "increment this number by 1 and give me the new value" — done atomically.
2. Base62 encode → `3d7`
3. Insert into DB: `("3d7", "www.abcd.com/sports/cricket/england/t20/stats")`
4. Return `https://short.ly/3d7`

### When someone visits `short.ly/3d7`:

1. Browser    → GET https://short.ly/3d7
2. Server     → Check Redis cache for key "3d7"
                 ├── HIT  → got long URL, skip to step 4
                 └── MISS → go to step 3
3. Server     → SELECT long_url FROM urls WHERE short_code = "3d7"
                 → returns "www.google.com/api/paste/new/cricket"
                 → also write it to cache for next time
4. Server     → HTTP 301 Redirect
                 Location: www.google.com/api/paste/new/cricket
5. Browser    → goes to Google

**301 vs 302:**
- `301` permanent → CDN caches forever, but no click analytics
- `302` temporary → every click hits your server (use for tracking)

---

## Database

| short_code (PK) | long_url               | created_at |
|-----------------|------------------------|------------|
| `3d7`           | `www.google.com/…`     | …          |

- KV store (DynamoDB, Cassandra, or Postgres with PK index)
- Shard by `short_code` (hash-based)

---

## Architecture

```
        Client
          │
     Load Balancer
          │
   ┌──────┴──────┐
   ▼             ▼
 Write          Read
 Service       Service
   │             │
   ▼             ▼
 Redis        Redis Cache
 INCR            │ miss
   │             ▼
   └──────▶  Database (KV)
                 │
                 ▼
                CDN
```

---



## Summary

Three ideas:

1. **Counter** for uniqueness
2. **Base62** for short URL-safe strings
3. **KV lookup + cache** because reads dominate

Same pattern applies to pastebins, image hosts, and share-link services.


# What is Sharding?

**Sharding = splitting one big database into smaller pieces (shards), each stored on a different server.**

You do it when one server can't hold all the data, or can't handle all the traffic, by itself.

---

## The Problem

Our URL shortener stores **180 billion rows**. One database server can't:
- Hold 90 TB of data
- Handle 120,000 reads/sec

We need many servers working together.

---

## The Idea

Split the table across, say, 10 servers. Each stores only **1/10 of the data**.

```
Shard 1 (Server A)  →  slice of short_codes
Shard 2 (Server B)  →  another slice
...
Shard 10 (Server J) →  another slice
```

When a request comes in for `short.ly/3d7`:
1. Figure out which shard owns `3d7`
2. Query only that one server

---

## How to Decide Which Shard Owns Which Row

The **sharding key** — in our case, `short_code`.

### Hash-based sharding (used here)

Hash the short code, mod by number of shards:

```
hash("3d7") % 10  =  7   →  Shard 7
hash("aB9") % 10  =  2   →  Shard 2
hash("xY1") % 10  =  4   →  Shard 4
```

Same code → same shard, every time. Deterministic and fast.

### Other strategies

- **Range-based** — `a*` to shard 1, `b*` to shard 2, etc. Risk: hot shards if traffic is uneven.
- **Geographic** — US users → US shard, EU users → EU shard.

---

## Why Hash-Based Works Well Here

- Short codes are essentially random → hash spreads them **evenly** across shards.
- No single shard becomes a hotspot.
- Each lookup goes to exactly one shard → predictable performance.

---

## Visualizing It

```
                ┌─────────────────┐
                │  Request:       │
                │  short.ly/3d7   │
                └────────┬────────┘
                         │
                         ▼
                 hash("3d7") % 10 = 7
                         │
        ┌────┬────┬──────┼──────┬────┬────┐
        ▼    ▼    ▼      ▼      ▼    ▼    ▼
      Shard Shard Shard Shard Shard Shard Shard
        1    2    3     ...    7    ...   10
                                │
                                ▼
                          Returns long URL
```

---


