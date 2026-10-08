# Bitly — URL Shortener

Source: hellointerview.com problem-breakdowns/bitly (free).

## Requirements

**Functional:** submit a long URL, get a short one back (optional custom alias, optional expiration); visiting the short URL redirects to the original.
**Out of scope:** auth, analytics.

**Non-functional:** short codes must be unique; redirect latency < 100ms; 99.99% availability (favor availability over consistency); scale to 1B shortened URLs, 100M DAU; **read:write ≈ 1000:1**, the single most load-bearing fact in this problem.

## Core entities

Original URL, Short URL, User.

## API

```
POST /urls  { long_url, custom_alias?, expiration_date? } → short_url
GET  /{short_code} → 302 redirect to original URL
```
302 (not 301) is deliberate — keeps control server-side, avoids browsers permanently caching the redirect, allows tracking/changing the mapping later.

## High-level design

Write path: validate → generate short code → insert mapping. Read path: look up short code → check expiration → 302. The whole problem is really two deep dives: how do you generate the code, and how do you make the read path fast enough for the 1000:1 skew.

## Deep dive 1 — short-code uniqueness

- **Bad:** take a prefix of the URL. Collides constantly.
- **Great option A — hash + base62:** SHA-256 the canonicalized URL, base62-encode, take the first ~8 chars. Base62 avoids URL-unsafe characters. Still needs a DB `UNIQUE` constraint + retry-with-salt on collision; sequential/derivable output is a minor downside.
- **Great option B — distributed counter + base62:** an atomic Redis `INCR` gives a strictly increasing ID, base62-encoded. Zero collisions by construction. 62^6 ≈ 56B >> 1B target, so 6 characters suffice. Downside: needs the counter to be highly available (Sentinel/Cluster), and sequential codes are enumerable/guessable. **Counter batching** (each writer claims a batch of e.g. 1000 values at once) cuts the Redis round-trip cost. Multi-region: give each region a disjoint counter range rather than coordinating a single global counter.

## Deep dive 2 — fast redirects at 1000:1 read skew

At 100M DAU × ~5 redirects/day that's ~5,800 req/s average, ~600k/s at a 100x spike — an index alone won't hold that.
- **Good:** a plain B-tree index / primary key on `short_code` — helps, but still bottlenecks under spike load.
- **Great option A — in-memory cache:** Redis/Memcached in front of the DB, cache-aside. Memory access (~100ns) vs. SSD (~0.1ms) is roughly a 1000x gap — this is the number that justifies the cache, not just the existence of read skew.
- **Great option B — CDN/edge:** serve the redirect itself from edge compute (Cloudflare Workers / Lambda@Edge) so popular codes never reach origin at all. Higher cost/complexity, but removes origin load entirely for hot links.

## Deep dive 3 — scaling to 1B URLs / 100M DAU

Storage: ~500 bytes/row × 1B ≈ 500GB — unremarkable, any mainstream DB is fine, write volume (~1 row/sec average) is not the bottleneck. Split into a Read Service and Write Service (independently, horizontally scalable, since their load profiles are wildly different). Counter service: centralized Redis with batching, HA via Sentinel/Cluster; DB `UNIQUE` constraint is the safety net if the counter ever loses uncommitted values.

## Final architecture

Client → Load Balancer → {Read Service (+Redis cache, optional CDN), Write Service (+Redis counter)} → Postgres (durable mapping store).

## Level expectations

- **Mid:** working shorten/redirect flow, recognizes uniqueness as a real requirement, proposes hashing or a counter, understands the 302 rationale, basic indexing; needs prompting to reach for caching.
- **Senior:** proactively surfaces both uniqueness approaches and the collision-vs-coordination tradeoff between them; discusses cache invalidation for expiring URLs; justifies DB choice; splits read/write services unprompted.
- **Staff+:** designs read-heavy from the start; unprompted multi-region counter-range design and Redis failover; flags security (enumerable codes) and product edges (custom-alias collisions, expiration cleanup) unprompted.
