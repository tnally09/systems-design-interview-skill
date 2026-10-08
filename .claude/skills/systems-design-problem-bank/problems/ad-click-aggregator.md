# Ad Click Aggregator

Source: hellointerview.com problem-breakdowns/ad-click-aggregator (free).

## Requirements

**Functional:** a user clicking an ad is redirected to the advertiser's site; advertisers query click metrics at 1-minute granularity.
**Non-functional:** peak 10k clicks/sec (~100M/day); sub-second query latency for analytics; fault-tolerant with no data loss; real-time-available metrics; idempotent click tracking (a given click counted exactly once).

## Core entities

ClickEvent (adId, userId, timestamp, impressionId), AggregatedMetrics (adId, minuteTimestamp, uniqueClicks), ImpressionId (a unique identifier minted per ad *impression*, not per click — this is what idempotency hangs on).

## API

```
POST /click  { impressionId, adId, userId }
GET  /metrics?adId=&start=&end= → aggregated counts
```

## High-level design (escalation)

- **Bad:** raw events in one database, `GROUP BY` at query time — falls over on both the aggregation cost and the write pressure.
- **Good:** raw events land in Cassandra, a periodic Spark batch job aggregates into an OLAP store (Redshift/Snowflake/BigQuery) — real latency between a click and it showing up in metrics.
- **Great:** click events go to a stream (Kafka/Kinesis); a stream processor (Flink) aggregates in real time using event-time semantics and watermarks (so out-of-order/late events still land in the correct minute bucket); results flush continuously to the OLAP store.

## Deep dive 1 — scaling to 10k clicks/sec

- Click Processor: stateless, horizontally scaled behind a load balancer.
- Stream: sharded by `adId` (Kafka's per-shard throughput cap, ~1,000 records/sec, is the actual number that forces sharding here).
- Stream processor: multiple Flink tasks, one or more per shard.
- OLAP store: auto-scaling if managed, or manually sharded by `advertiserId` if self-managed.
- **Hot-shard fix for a viral ad:** append a random suffix to the partition key (`adId:0..N`) to spread one ad's clicks across multiple shards; Flink strips the suffix before the final aggregation step and sums across the split shards back into one number.

## Deep dive 2 — preventing data loss

- Stream retention (7 days on Kafka/Kinesis) means a failed downstream processor can simply replay from where it left off.
- Flink checkpoints its aggregation state to S3 periodically — most valuable for larger aggregation windows; less critical at the 1-minute granularity this problem actually needs, but worth naming as a general mechanism.
- **Reconciliation (a Lambda-architecture pattern):** raw events are also dumped to a data lake (via Kafka Connect/Kinesis Firehose to S3); a daily Spark batch job re-aggregates everything from scratch and is diffed against the real-time (Flink) results to catch correctness drift the streaming path might have introduced. This is a *correctness* safety net, distinct from the fault-tolerance mechanisms above — it's designed to catch bugs/edge cases in the speed layer, not just process crashes.

## Deep dive 3 — idempotency & click-fraud resistance

- **Bad:** require login and dedup by `(userId, adId)` — breaks legitimately for retargeting, where the same user clicking the same ad again in a new context should count.
- **Great:** the Ad Placement Service mints a unique `impressionId` per ad *instance shown* (not per click), HMAC-signs it, and hands it to the browser; the Click Processor verifies the signature and checks an idempotency cache before accepting the click — but writes to the stream *first*, then updates the idempotency cache, so a cache failure can't cause data loss (at worst, a very rare double-count, which is the safer failure direction than silently dropping a click).

## Deep dive 4 — low-latency queries over large time windows

Minute-level pre-aggregation handles most queries directly. For queries spanning days/weeks, a nightly cron job additionally rolls the minute-level data up into daily/weekly pre-aggregated tables, so a wide-range query never has to scan and sum thousands of minute buckets live.

## Final architecture

Click → Click Processor (HMAC/impression verification, idempotency cache) → Kafka (sharded by adId, with salting for hot ads) → Flink (windowed aggregation by adId+minute, checkpointed) → OLAP store ← advertiser queries. In parallel: raw events → S3 → nightly Spark reconciliation job.

## Level expectations

- **Mid (E4):** understands why pre-aggregation is necessary at all, proposes the batch-processing ("Good") version, handles direct probing on idempotency and database choice reasonably.
- **Senior (E5):** discusses the batch-vs-real-time tradeoff explicitly, anticipates hot-shard scaling issues before being asked, justifies each technology choice, shows real depth in at least one deep dive.
- **Staff+ (E6+):** genuine expertise across a large share of the topics, proactively identifies and solves problems, teaches the interviewer something, comfortable discussing complex fault-tolerance/correctness scenarios like the reconciliation job.
