# Facebook Post Search

Source: hellointerview.com problem-breakdowns/fb-post-search (free).

## Requirements

**Functional:** create and like posts; search posts by keyword; sort results by recency or like count.
**Out of scope:** fuzzy matching, personalization, privacy rules, sophisticated relevance ranking, media, real-time result updates.

**Non-functional:** median query latency < 500ms; high request volume; a new post searchable within < 1 minute; all posts eventually discoverable (older posts may lag); high availability.

## Core entities

User, Post (searchable content, timestamp, like count), Like (mostly matters as an aggregate count, not individually).

## API

```
POST /posts
POST /posts/{id}/like
GET  /search?keyword=&sort={recency|likes}
```

## High-level design

**Write path:** an ingestion service tokenizes each post and writes post IDs into Redis-backed inverted indexes — kept as two separate indexes, one ordered/queryable by recency and one by like count, rather than one index re-sorted per query.
**Read path:** API Gateway → Search Service → the appropriate Redis inverted index → Post Service to hydrate the actual post content for the matched IDs.

The core mechanism: an inverted index maps a keyword to the set of post IDs containing it, so a query never scans post content at request time — it's a direct dictionary lookup instead.

## Deep dive 1 — high request volume

- **Bad:** un-indexed database scans per query.
- **Good:** a distributed cache alongside the search service, short (<1 min) TTL matching the "new posts searchable within a minute" requirement.
- **Great:** CDN edge caching with cache-control headers, getting response times down to ~10ms for the (large) share of queries that repeat or overlap.

## Deep dive 2 — multi-keyword queries

- **Bad:** fetch each keyword's full post-ID list and intersect/filter at request time — for common keywords this means megabyte-scale payloads and millions of comparisons per query.
- **Good:** set intersection of the keyword lists, then filter the resulting (smaller) post set by content — better, but still real work for common-keyword pairs.
- **Great:** pre-index common bigrams/shingles (e.g. "Taylor Swift" as a single compound key) so a common multi-word phrase resolves as one direct lookup instead of an intersection; fall back to runtime intersection only for uncached/uncommon phrase combinations.

## Deep dive 3 — write volume from likes

- **Bad:** write an index update on every single like event — the index becomes a write bottleneck under viral-post conditions.
- **Good:** batch like-count updates over a 30-second window before touching the index.
- **Great — two-stage:** update the index only at logarithmic milestones (powers of 2 — 1, 2, 4, 8, 16…) rather than every batch window, then re-rank the top N×2 candidates with a fresh read straight from the Like Service at query time. This keeps index writes rare (proportional to log of the like count, not the like count itself) while still surfacing accurate top results.

## Deep dive 4 — storage growth of the inverted index

Cap how many post IDs a single keyword's index entry can hold (e.g. 1k-10k); once a keyword's index would exceed that, move its rarely-accessed tail into cold storage (S3/R2) and query it only as a fallback after checking the hot Redis index first.

## Final architecture

API Gateway → Search Service (cached) → Redis (hot inverted indexes, bigram-indexed) + S3 (cold overflow indexes) → Post Service; Kafka streams post-creation and like events to partitioned ingestion services that maintain the indexes.

## Level expectations

- **Mid (80% breadth / 20% depth):** covers API design, data model, and both the ingestion and query paths at a basic level; expect to be guided toward the deeper optimizations.
- **Senior (60% breadth / 40% depth):** moves quickly through the basics to deep-dive the critical paths; must land on the inverted-index + caching strategy with real tradeoff reasoning, not just naming it.
- **Staff+ (40% breadth / 60% depth):** proactively identifies and solves the harder issues (write amplification from likes, storage growth) unprompted, covers multiple deep dives with genuinely novel solutions.
