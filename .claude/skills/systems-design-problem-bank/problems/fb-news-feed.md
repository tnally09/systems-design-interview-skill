# Facebook News Feed

Source: hellointerview.com problem-breakdowns/fb-news-feed (free).

## Requirements

**Functional:** create posts; follow/friend other users; view a reverse-chronological feed; page through it.
**Out of scope:** likes, comments, privacy controls.

**Non-functional:** availability over consistency (up to ~1 minute of staleness acceptable); < 500ms feed load; scale to 2B users; unlimited follow counts (a load-bearing constraint — no cap on either side).

## Core entities

User, Follow (uni-directional), Post.

## API

```
POST /posts → postId
PUT  /users/:id/follow → 200
GET  /feed?pageSize=&cursor= → { items: Post[], nextCursor }
```

## High-level design

Post creation is a plain stateless write. Follow relationships live in a table keyed `(follower, followee)` with a GSI for the reverse lookup. The interesting problem is entirely in how the feed itself gets assembled — that's the whole deep-dive section.

## Deep dive 1 — huge follow counts (fan-out on read)

- **Bad:** compute the feed at read time by querying Follow then Post per followee — multiple round trips, cascades badly under load.
- **Good:** a `PrecomputedFeed` table holding ~200 recent posts per user, fanned out on write. 2KB/user × 2B users ≈ 4TB — a reasonable storage cost. Falls back to the naive on-read approach for pagination past 200.
- **Great:** the same precomputation, plus a follow-count cap (production Facebook uses ~5,000) so no single user's fan-out work is unbounded, accepting the stated eventual-consistency window.

## Deep dive 2 — huge follower counts (fan-out on write)

- **Bad:** blast a write to every follower's feed synchronously on post — for a celebrity account this is millions of writes in one request; connection limits alone make it fail.
- **Good:** queue the fan-out job (SQS) and let a worker fleet drain it asynchronously, meeting the 1-minute staleness bound. Problem: job size varies wildly (1K followers vs. 1M), so worker load is lumpy.
- **Great — hybrid fan-out:** accounts above a follower threshold are flagged "non-precomputed" and skipped by the fan-out workers entirely; the Feed Service merges each user's precomputed feed with the *recent* posts of any high-follower accounts they follow, computed at read time. This bounds both the write-side fan-out cost and the read-side merge cost, with a tunable threshold.

## Deep dive 3 — hot posts (uneven read pattern)

- **Bad:** read straight from the Post table — a viral post concentrates reads on one partition/shard.
- **Good:** a sharded Redis cache in front of Post — but a viral post is still, by definition, a hot key that overwhelms whichever shard owns it.
- **Great — replicated (not sharded) cache:** every cache instance can serve every post independently; the load balancer spreads read traffic across all N instances, so a viral post's reads spread across all of them instead of hammering one shard. Accepts a slightly higher aggregate cache-miss rate in exchange for eliminating the hot-shard problem, with no coordination needed between instances.

## Final architecture

API Gateway → Post Service (DynamoDB + replicated Redis cache) + Follow Service (DynamoDB with reverse GSI) + Feed Service (PrecomputedFeed table + SQS-driven fan-out workers, merging in non-precomputed high-follower accounts at read time).

## Level expectations

- **Mid (E4):** clean API/data model, functional design, some "Good" solutions land; may not cover every scaling edge; interviewer drives later stages.
- **Senior (E5):** ~60% depth / 40% breadth; detailed on 2+ of the deep dives, proactively identifies the fan-out bottlenecks and covers both directions (read-fanout and write-fanout) without being walked to them.
- **Staff+ (E6+):** ≥60% depth; covers all three deep dives plus extra optimizations, draws on real production experience, minimal interviewer steering.
