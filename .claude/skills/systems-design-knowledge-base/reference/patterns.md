# Common Patterns

Hellointerview names 8 recurring patterns. Their individual deep-dive pages are paywalled, so this file combines their free one-line framing (from the "in a hurry" patterns page) with concrete examples pulled from the 15 free problem breakdowns in [[systems-design-problem-bank]] — which is where most of the real teaching content actually is. Read the referenced problem file for full detail on any example.

## Scaling Reads

**Problem:** read volume far exceeds what a single database can serve directly. **Toolkit:** indexing, read replicas, caching layers.
- Bitly: 1000:1 read/write ratio → in-memory cache (short_code → long_url) or CDN/edge redirect for the hottest links, layered on top of a plain indexed lookup.
- fb-news-feed: precomputed per-user feed table (fan-out on write) traded against a hybrid that skips precomputation for accounts with huge follower counts.
- fb-post-search: inverted index in Redis, CDN caching at the edge for non-personalized queries.
- **Common escalation ladder across problems:** un-indexed scan (bad) → indexed query (good, but caps out) → cache in front of the index (great) → CDN/edge for the hottest, non-personalized subset (great+, only when applicable).

## Scaling Writes

**Problem:** write volume exceeds one instance's throughput. **Toolkit:** sharding, batching, load shedding. Per `reference/core-concepts.md`'s sharding section: establish the need with real numbers before reaching for this.
- top-k-videos (YouTube views): 700k writes/sec → shard by video ID across 70+ shards, and batch/aggregate in Flink with tumbling windows rather than writing every raw event.
- ad-click-aggregator: 10k clicks/sec → stream to Kafka sharded by AdId, aggregate in Flink, flush aggregates (not raw events) to the OLAP store; salted keys for hot ad IDs.
- web-crawler: 10B pages/5 days → parallelize across machines, not a single write path at all.

## Real-Time Updates

**Problem:** the client needs to see server-side changes without polling. **Toolkit:** WebSockets (bidirectional), SSE (server→client only), or plain polling — chosen by the actual traffic shape, per `reference/core-concepts.md`'s networking section.
- fb-live-comments: SSE, specifically because the traffic is asymmetric (many viewers, few commenters) — WebSockets would be the wrong, heavier default here.
- whatsapp: WebSocket-based bidirectional protocol, justified because chat is genuinely bidirectional and high-frequency, plus heartbeats for fast dead-connection detection.
- Ticketmaster: SSE for live seat-map updates, escalating to a Redis-backed virtual waiting queue for the mega-event case where even SSE gets overwhelmed.

## Managing Long-Running Tasks

**Problem:** an operation takes longer than a synchronous request should hold open. **Toolkit:** decouple submission from execution — queue + async worker + a status check or callback, never a held-open request.
- LeetCode judge: submission goes on a queue (SQS), workers execute in isolated containers, client polls `GET /check/:id`.
- YouTube upload: post-processing is a DAG (segment → transcode → manifest) run by workers, orchestrated (e.g. Temporal), triggered by S3 event notifications between stages.
- Uber driver-request timeout/retry: a durable execution framework (Temporal/Step Functions) that survives service crashes, rather than an in-memory timer.

## Dealing with Contention

**Problem:** multiple actors race for the same resource. **Toolkit:** locks (with TTL), optimistic concurrency, or serializing through a queue.
- Ticketmaster seat reservation: escalates from a bad long-held DB lock → a good status+expiration+cron approach → a great Redis distributed lock with TTL (`SET key value NX EX seconds`) backed by a DB-level optimistic check as the safety net.
- Tinder mutual-swipe matching: a great solution uses Redis Lua scripts for atomic check-and-record, or a Cassandra partition key combining both user IDs so a mutual match lands in one partition and can use a single-partition transaction.
- Uber driver assignment: Redis distributed lock with a short (10s) TTL matching the accept window, so failure/timeout self-heals without manual cleanup.
- gopuff inventory: chose a single Postgres ACID transaction over a distributed lock specifically to avoid the failure modes (orphaned locks, deadlocks) that locks introduce — a reminder that "add a distributed lock" isn't always the answer; sometimes a plain transaction on the data's real home is the great solution (see Shopify in `reference/in-the-wild.md` for the production-scale version of this same point).

## Handling Large Blobs

**Problem:** large files shouldn't be proxied through your application servers. **Toolkit:** presigned URLs for direct client↔blob-storage transfer, a CDN for delivery, chunking for anything too large for one request.
- Dropbox: presigned URLs for direct-to-S3 upload/download; chunked (5-10MB) + fingerprinted (SHA-256) multipart upload for resumability, verified server-side via S3's ETag/ListParts rather than trusting client-reported chunk status.
- YouTube: same presigned-URL and multipart pattern for upload; segmented, multi-format storage plus a manifest file is what actually enables adaptive-bitrate streaming on playback.

## Multi-Step Processes

**Problem:** a workflow spans multiple services/steps and must survive partial failure without leaving inconsistent state. **Toolkit:** orchestration (explicit DAG/workflow engine) or choreography (event-driven), retries, idempotency.
- YouTube post-processing DAG: explicit dependency graph (split → transcode → manifest → mark-complete), parallelized where the DAG allows, intermediate artifacts in S3 so workers don't need to talk to each other directly.
- ad-click-aggregator's Lambda-architecture reconciliation: a nightly batch (Spark) recomputation of the same aggregates the real-time path (Flink) produces, diffing the two to catch correctness drift — a multi-step *correctness* pattern, not just a workflow-orchestration one.

## Proximity-Based Services

**Problem:** "nearest N" queries over 2D location data. **Toolkit:** see `reference/deep-dives.md`'s Proximity Search entry for the full mechanism (spatial trees vs. encoded keys) — the pattern-level point is just: don't index lat/long naively, always produce a candidate set and post-filter by exact distance.
- Uber driver matching: Redis geospatial (GEOADD/GEOSEARCH, geohash-backed) with TTL-based cleanup of stale/offline drivers, plus adaptive client-side ping frequency so the write volume itself stays manageable.
- Tinder feed: geospatial filtering combined with Elasticsearch for the broader multi-attribute (age, interest, distance) query, since pure proximity isn't the only filter in play.
- gopuff nearby-DC lookup: cheap radius prune first (Haversine), then only call the expensive real travel-time service for the small filtered candidate set — the two-phase pattern applied to a non-geohash context.
