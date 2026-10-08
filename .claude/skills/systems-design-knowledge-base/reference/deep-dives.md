# Technology Deep Dives

Condensed from hellointerview's 8 free technology deep-dive pages. Each entry: what it actually is, when it's the right call, when it isn't, and the specific mechanisms worth knowing for a deep dive (not just the name).

## Redis

- **What it is:** single-threaded (per command) in-memory data structure store — strings, hashes, lists, sets, sorted sets, streams, geospatial indexes. Microsecond operations, ~100k writes/sec on a single node.
- **Durability caveat to know:** acknowledged writes can still be lost (async replication/persistence) — don't reach for Redis where you need a hard durability guarantee.
- **Interview use cases and the mechanism that makes each work:**
  - Caching — TTL + eviction (LRU default).
  - Distributed locks — atomic ops via Lua scripts or `SET key value NX EX seconds`; know that naive Redlock is controversial, don't present it as an uncontested solution.
  - Leaderboards — sorted sets.
  - Rate limiting — counters (fixed window) or sorted sets (sliding window).
  - Proximity search — GEOADD/GEOSEARCH (geohash-backed); see `proximity-search` below and `reference/patterns.md`.
  - Event sourcing/light queueing — streams with consumer groups.
  - Pub/Sub — real-time fan-out, at-most-once delivery only (messages aren't persisted — don't use this where delivery must be guaranteed).
- **Scaling:** single node, replicas, or Redis Cluster (16,384 hash slots). No built-in query router — the client/app must know how to address the right shard.
- **When it's the wrong call:** you need durability guarantees, your working set exceeds RAM, you need complex queries/joins, or you need durable replay for multiple independent consumers (that's Kafka's job, not Redis's).
- **Hot key problem:** one key gets disproportionate traffic — mitigate with client-side caching of that key, key duplication across nodes, or read replicas. This is the generic hot-key pattern that recurs across nearly every problem breakdown.

## Elasticsearch

- **What it is:** a document store built around an inverted index (word → document IDs) for full-text and faceted search, plus geospatial search via geo_point/geo_shape fields (BKD trees + geohashes).
- **Architecture worth knowing for a deep dive:** documents live in indices, indices are split into shards, each shard is a Lucene index made of immutable segments. Immutability is *why* it's fast to write, cache, and compress — inserts create new segments, deletes/updates are soft (marked, cleaned up on background merge).
- **When to use:** full-text search, geospatial queries, faceted filter/sort combinations, denormalized read-heavy workloads.
- **When NOT to use:** small datasets (< ~100k docs — the primary DB's index is enough), write-heavy workloads (soft-delete/update overhead degrades it), as a primary/authoritative store (it's a secondary index, not a system of record), or anywhere strong consistency is required (it's eventually consistent by design).
- **The sync problem is the real interview content, not the query DSL:** keep it in sync with the source of truth via Change Data Capture, and be explicit that results can be briefly stale. Denormalize into the document so a query resolves in 1-2 round trips rather than needing joins back to the primary store.

## Kafka

- **What it is:** a distributed, partitioned, append-only log. Topics are logical groupings of ordered partitions; producers write, consumers pull and track position via offsets; a message's key determines its partition (`hash(key) % partitions`), which is what gives you per-key ordering.
- **Two interview framings:** as a message queue (decouple producer/consumer rate, e.g. async video transcoding, ordered virtual-queue processing) or as a stream (multiple independent consumers, replay).
- **Key mechanism for interviews: hot partitions.** A single popular key (viral post, popular driver region) overloads one partition. Fixes: accept default random partitioning (loses ordering), salt the key, use a compound key, or apply producer-side backpressure. This shows up nearly identically across Ticketmaster's booking flow, ad-click-aggregator, and fb-post-search.
- **Delivery guarantees:** at-least-once by default; exactly-once needs idempotent producers + transactions. Unlike SQS, Kafka gives you no built-in retry topic/DLQ — you build that yourself on the consumer side.
- **Rough numbers to reason with, not cite as gospel:** a single broker handles on the order of ~1TB storage and ~1M messages/sec; scale with more brokers and partitioning, not by cramming large payloads through Kafka itself (store blobs in S3, pass a pointer).
- **One-line framing hellointerview uses:** "Kafka is always available, sometimes consistent" — partition strategy and hot-partition handling are the actual interview content, not the existence of Kafka.

## API Gateway

- **What it is:** the single entry point in front of a microservices architecture — the hotel-front-desk analogy: clients shouldn't need to know which internal service handles what.
- **What it actually does, in order:** request validation → middleware (auth, rate limiting, SSL termination, CORS, versioning) → routing (by path/method/header) → possible protocol translation to the backend → response transformation → optional caching of non-personalized responses.
- **The most common interview miss:** candidates jump straight to listing middleware and skip the primary job, which is routing.
- **Scaling:** stateless, scales horizontally behind a load balancer; multi-region via GeoDNS with synchronized config.
- **When to use:** microservices architectures. **When to skip:** a simple monolith doesn't need one.
- **Calibration:** this is a low-investment component in most interviews — state its responsibilities and move on, similar to API design's "don't over-invest" guidance.

## Cassandra

- **What it is:** a wide-column, write-optimized, AP-leaning (tunable consistency) distributed database, originally from Facebook. Keyspace → Table → Row (identified by a primary key) → Columns (can vary per row).
- **Primary key = partition key (which node/partition) + optional clustering key (sort order within the partition).** Query-driven modeling: design tables around your access patterns, not normalized entities — denormalize across multiple tables rather than join, because Cassandra doesn't support joins or ad-hoc aggregation.
- **Storage engine (LSM trees) is the deep-dive content:** writes go to a commit log (durability) + memtable (in-memory sorted), memtables flush to immutable SSTables, background compaction merges them and clears tombstones. This is why Cassandra is write-optimized — it never does an in-place update.
- **Partitioning:** consistent hashing with virtual nodes (vnodes), same principle as `reference/core-concepts.md`'s consistent hashing section.
- **Tunable consistency:** per-query consistency level (ONE/QUORUM/ALL) on reads and writes; QUORUM+QUORUM gives you read-your-writes without going fully strong.
- **Real modeling examples worth citing:** Discord partitions messages by `(channel_id, time-bucket)` with `message_id DESC` clustering to prevent one busy channel's partition from growing unbounded (see `reference/in-the-wild.md`); a Ticketmaster-style ticket table can partition by `(event_id, section_id)` with `seat_id` clustering, plus a separate denormalized aggregate table for section-level counts.
- **When NOT to use:** you need joins, ad-hoc aggregation, strict consistency, or ACID transactions across rows (Cassandra gives row-level atomicity only).

## DynamoDB

- **What it is:** AWS's managed key-value/document store. Table → Items (≤ 400KB) → Attributes, schema-less. Partition key (hash-distributed) + optional sort key (B-tree ordered within partition, enables range queries).
- **Secondary indexes:** GSI (different partition+sort key, own partitions, eventually consistent only, up to 20) vs LSI (same partition key, different sort key, co-located, can be strongly consistent, must be defined at table creation, capped at 10GB per partition key). Use GSIs to query by a non-primary attribute across all partitions; LSIs for range queries within a partition you already know.
- **Consistency is chosen per-request, not per-table:** eventual (default, cheaper, lower latency) vs strong (double the read cost, base table/LSI only — not available on GSIs). Supports ACID transactions via TransactWriteItems/TransactGetItems (up to 100 items).
- **Under the hood** (worth knowing for a deep dive): Multi-Paxos, leader-based replication across 3 AZs; strongly consistent reads go to the leader.
- **Capacity numbers:** a single partition supports ~3,000 RCU / 1,000 WCU — roughly 12MB/s reads or 1MB/s writes per partition; this is what forces a good partition key at real scale.
- **DAX** — an in-memory accelerator cache (microsecond reads) sitting in front, read-through/write-through, but doesn't cache strongly-consistent reads and can miss updates made outside DynamoDB's own API.
- **Streams** — DynamoDB's CDC mechanism, used to keep Elasticsearch/analytics/derived stores in sync.
- **When NOT to use:** you need SQL-style joins/ad-hoc aggregation, extremely high sustained write volume makes the pricing model prohibitive, or you find yourself needing many GSIs/LSIs just to support your query patterns (a sign a relational model fits better), or you can't accept AWS lock-in.

## Proximity Search

- **The actual problem:** B-tree indexes are one-dimensional; lat/long is two-dimensional, so a naive index (or even a composite one) can't preserve 2D closeness — you fall back to scanning. Every real solution below produces a *candidate set*, then post-filters by exact distance — that two-phase framing is the answer interviewers actually want, more than the name of a specific structure.
- **Spatial trees (for geometric/shape data — polygons, roads, delivery zones; PostGIS default):**
  - Quadtree — recursively quarters the map; adapts to density but gets uneven depth and is pointer-heavy (disk-unfriendly) in dense areas.
  - k-d / BKD tree — splits alternating dimensions at the median, so depth stays balanced regardless of density; BKD packs points into disk-page blocks (Elasticsearch's geo field implementation) but is essentially write-once/batch-built.
  - R-tree — bounding-rectangle hierarchy, handles shapes (not just points), balances like a B-tree so it's update-friendly; the production default for disk-based geospatial (PostGIS/SQLite/Oracle Spatial).
- **Encoded keys (for moving point data — drivers, users, devices; cheap single-key writes on a standard B-tree index):**
  - Geohash — recursive grid, each added character is ~5x finer; shared prefix = spatial locality, so a prefix scan on a normal string index works. Boundary caveat: nearby points across a cell edge get different prefixes — fix by querying the cell plus its 8 neighbors, then post-filter by exact distance.
  - S2 (Google) — spherical, roughly equal-area cells globally (geohash distorts near the poles), 64-bit hierarchical IDs; underlies MongoDB's 2dsphere index.
  - H3 (Uber) — hexagonal cells (uniform 6-neighbor distance, unlike a square's mixed edge/corner neighbors); IDs aren't a space-filling curve like geohash/S2, so proximity is computed via cheap ring math rather than prefix matching. Uber's dispatch pattern: snap drivers to ~200m H3 cells, query the rider's cell plus surrounding rings, post-filter by exact distance — enables millions of location writes/sec.
- **Interview calibration:** naming geohash vs. S2 vs. H3 scores less than correctly explaining *why plain lat/long indexing fails* and then picking a boring, defensible option (Redis GEOADD/geohash for most interviews) with the two-phase candidate-set-then-filter reasoning intact.

## Time-Series Databases

- **When a TSDB actually earns its keep:** very high write throughput of append-only, timestamped, tag-labeled numeric data with low-cardinality tags and small deltas between consecutive values (e.g., server metrics: 100k servers × 5 metrics × every 10s = 50k writes/sec).
- **The mechanisms that make this work, worth citing in a deep dive:**
  - Append-only writes (sequential I/O, not random) + LSM-tree storage (memtable → immutable SSTable → background compaction), same family as Cassandra's engine.
  - Delta / delta-of-delta encoding for timestamps and XOR-based compression for floats — exploits how similar adjacent readings are; can get well under 2 bytes/point vs. 50-100 bytes naively.
  - Time-based partitioning — writes localize to the "now" partition, reads target only relevant partitions, retention becomes "drop old partitions" instead of a DELETE scan.
  - Bloom filters per SSTable to skip files that provably don't contain a queried series with zero I/O.
  - Downsampling/rollups — keep full resolution briefly, coarsen older data (e.g., 24h full-res → 7d at 1-min avg → 30d at 5-min avg → 1yr at 1hr avg), and pre-aggregate so most queries never touch raw data.
- **Data model:** measurement (like a table) + tags (indexed, low-cardinality, used for filtering) + fields (unindexed values) + timestamp. High-cardinality identifiers (user IDs, request IDs) must be fields, never tags — this is the cardinality trap.
- **The explicit warning hellointerview gives: don't reach for a TSDB just because the data has timestamps.** A Top-K-style aggregation across millions of distinct series (e.g., YouTube's top-k videos problem) can perform *worse* on a TSDB than a purpose-built streaming aggregation (Flink) + cache, because sorting/ranking across huge cardinality violates the TSDB's core assumption of low-cardinality tags. Treat "it has timestamps" as insufficient justification on its own — establish the actual access pattern first, same discipline as every other specialized-technology choice in this file.
