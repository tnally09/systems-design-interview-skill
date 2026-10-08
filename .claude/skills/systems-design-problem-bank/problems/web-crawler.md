# Web Crawler (for LLM training data)

Source: hellointerview.com problem-breakdowns/web-crawler (free).

## Requirements

**Functional:** crawl the web starting from seed URLs; extract and store text content.
**Out of scope:** text processing/cleanup, non-text data, dynamic (JS-rendered) content, authenticated pages.

**Non-functional:** fault-tolerant (no lost progress on failure); polite (respect `robots.txt`, avoid overloading any single server); complete 10B pages within 5 days.
**Out of scope:** security, cost optimization, legal compliance.

## Core entities

URL (status, crawl depth, last-crawled time — in a metadata store), Domain (robots.txt rules, last-crawl time), content hash (for dedup), raw HTML/extracted text (blob storage).

## System interface

Input: seed URLs. Output: extracted text per crawled page.

## High-level design

A Frontier Queue holds URLs waiting to be crawled → Crawlers fetch (DNS resolve + HTTP GET) → Parsers extract text and discover new links, feeding them back into the frontier → text goes to S3, metadata to a fast key-value store (DynamoDB).

## Deep dive 1 — fault tolerance without losing progress

- **Bad:** in-memory retry timers — all state (and all in-flight progress) vanishes if a crawler process dies.
- **Good:** Kafka with hand-rolled exponential backoff — workable, but the backoff/retry logic has to be built and maintained entirely by hand.
- **Great:** SQS, using its visibility-timeout primitive (`ChangeMessageVisibility`) to build backoff cleanly even though SQS has no built-in exponential-backoff setting — a message stays invisible-but-undeleted until a crawler explicitly finishes with it, so a crashed crawler's in-flight URLs simply become visible again for another worker, with zero custom state to reconstruct.

## Deep dive 2 — politeness / robots.txt

- **Bad:** ignore crawl-delay and rate limits entirely.
- **Good:** per-domain rate limiting with random jitter, so many crawlers don't synchronize into a burst against the same domain.
- **Great:** cache each domain's `robots.txt` (don't refetch it every request); a Redis-based per-domain lock enforcing that domain's specific crawl-delay; a global sliding-window rate limit (e.g. 1 req/sec) as a floor; a crawler that isn't yet allowed to hit a domain defers its own message via `ChangeMessageVisibility` rather than busy-waiting or dropping the URL.

## Deep dive 3 — scaling to 10B pages in 5 days

**Capacity math as the actual design driver (a case where hellointerview's own problem breakdown does the math, because the decision genuinely turns on it):** a network-optimized instance can push roughly 200 Gbps; at a realistic ~30% utilization that's about 3,750 pages/sec per machine; hitting the 5-day target needs on the order of 8 such machines running in parallel across many domains (parallelism across domains is what makes this feasible without violating per-domain politeness limits).

**Efficiency mechanisms that matter at this scale:**
- URL-level dedup: check the metadata store before ever queuing a URL.
- Content-level dedup: hash page content; check via an indexed metadata lookup or a Bloom filter (e.g. RedisBloom) for a cheaper first pass.
- Crawler-trap avoidance: cap crawl depth (e.g. 15-20 hops from a seed) so infinite-link-generating pages can't consume the whole budget.
- DNS: cache resolutions and spread across multiple resolvers — early profiling cited in the source found DNS lookups consuming up to 70% of thread time when unoptimized, which is why this gets called out specifically rather than assumed to be free.
- Parser fleet: auto-scales off the depth of the downstream processing queue (Lambda/Fargate), decoupled from crawler fleet scaling.

## Final architecture

Frontier Queue (SQS, with backoff + DLQ) → Crawler fleet (parallel, DNS-cached, per-domain rate-limited via Redis locks) → S3 (raw HTML/text) + DynamoDB (URL/domain/robots.txt metadata) → dedup via hash + Bloom filter → auto-scaled Parser fleet.

## Level expectations

- **Mid (E4):** describes the high-level data flow, basic politeness handling, surface-level scaling discussion; interviewer probes the basics and the candidate drives early stages but expects guidance later.
- **Senior:** real depth on queueing technology choice (SQS vs. Kafka, and why), detailed retry/rate-limiting mechanics, a genuine scaling analysis with bottlenecks identified rather than asserted, proactive tradeoff articulation.
- **Staff+:** 3+ deep dives with real depth, technology choices clearly grounded in practical experience, advanced distributed-systems fluency, the interviewer should come away with a new perspective.
