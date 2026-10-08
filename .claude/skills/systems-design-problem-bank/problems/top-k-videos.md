# YouTube Top-K Videos — Real-Time View Leaderboard

Source: hellointerview.com problem-breakdowns/top-k (free). Not to be confused with `youtube.md` (upload/streaming) — this is a narrower analytics/aggregation problem sharing the YouTube setting.

## Requirements

**Functional:** query the top K (≤1,000) videos by view count, for all-time and for tumbling windows (last hour/day/month).
**Out of scope:** arbitrary custom time ranges, historical (non-current) lookback windows.

**Non-functional:** ≤ 1 minute delay from a view event to it affecting the ranking; exact (not approximate) counts as the starting bar; response in tens of ms; ~700k views/sec (70B views/day); ~3.6B total videos.

## Core entities

Video, View (event), Time Window (query parameter).

## API

```
GET /views/top-k?window={ALL|1H|1D|1M}&k={K} → { videoId, views }[]
```

## High-level design (baseline)

Kafka stream of view events, partitioned by video ID → consumer increments counters in Postgres → Top-K service runs an indexed, sorted query. **Immediately acknowledged bottleneck:** 700k writes/sec is well beyond what a relational DB handles as individual row increments.

## Deep dive 1 — cut read load

- **Good:** cache the top-K result with a TTL, recompute on miss.
- **Great:** precompute on a schedule (cron) and proactively warm the cache before the old value expires, so reads never see a cold-cache spike.

## Deep dive 2 — handle the write throughput

- **Sharding:** partition the counter store by video ID across ~70+ shards.
- **Batching (the real fix):** don't write per-event at all — aggregate in Flink using tumbling windows (e.g. hourly) and batch-flush the aggregates, not raw events.

## Deep dive 3 — make the top-K query itself cheap

- **Good:** maintain separate aggregate tables per granularity (hour/day/month) instead of aggregating on the fly.
- **Great:** one table per window (`VideoViewsLastHour`, etc.) indexed on the views column for an O(k) top-K read; alternatively compute the aggregates in-memory in Flink (RocksDB state backend) and only materialize the result.

## Deep dive 4 — sliding (not just tumbling) windows

Track minute-grained buckets, increment on new views and decrement as the oldest minute rolls out of the window. Real cost: requires reading historical buckets and a larger storage footprint than a pure tumbling-window design — a legitimate complexity tradeoff to name, not a free upgrade.

## Deep dive 5 — approximate techniques (when exactness isn't actually required)

- **Good:** Redis Count-Min Sketch + sorted sets — cheap, but Redis's durability caveats apply (see `systems-design-knowledge-base/reference/deep-dives.md`).
- **Great:** a Flink-native Count-Min Sketch implementation with checkpoint-based recovery, avoiding the external-store durability question entirely.

## Deep dive 6 — why not just use a specialized DB?

- **InfluxDB/Prometheus:** poor fit — billions of distinct `videoId` values is exactly the high-cardinality case time-series DBs choke on (full scans instead of the intended fast path).
- **TimescaleDB:** workable — hypertables + continuous aggregates mirror the hand-built solution, but still benefits from a cache in front.
- **Real-time OLAP (Druid/Pinot/ClickHouse):** workable — ingest-time rollups achieve the same pre-aggregation goal natively.
This whole deep dive is the concrete illustration of `systems-design-knowledge-base`'s warning: "having timestamps" doesn't mean a time-series database is the right tool — cardinality is what actually decides it here.

## Final architecture

View events → Kafka (partitioned by video ID) → Flink (windowed aggregation, batches writes) → precomputed aggregate tables/state → Redis cache (proactively warmed) → API.

## Level expectations

- **Mid (80% breadth / 20% depth):** end-to-end solution that identifies the major bottleneck (write throughput); surface-level tool familiarity; interviewer drives most of the problem-solving.
- **Senior (60% breadth / 40% depth):** near-optimal design, proactively resolves bottlenecks, understands the tradeoffs between the options (e.g. exact vs. approximate, tumbling vs. sliding) rather than picking one by default.
- **Staff+ (40% breadth / 60% depth):** sees the full solution space and picks with clear judgment, anticipates the cardinality trap before being asked, minimal steering needed.
