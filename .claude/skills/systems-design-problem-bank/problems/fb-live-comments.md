# Facebook Live Comments

Source: hellointerview.com problem-breakdowns/fb-live-comments (free).

## Requirements

**Functional:** post a comment on a live video; viewers see new comments near-real-time; viewers can load historical comments from before they joined.
**Non-functional:** scale to millions of concurrent live videos, thousands of comments/sec on a single popular stream; availability over consistency (eventual consistency acceptable); < 200ms end-to-end for the update to feel "real-time."

## Core entities

User (viewer or broadcaster), Live Video, Comment.

## API

```
POST /comments/:liveVideoId
GET  /comments/:liveVideoId?cursor=&pageSize=&sort=desc
```
Auth via a token in headers, not the request body — user identity is never client-asserted.

## High-level design

Commenter Client → Comment Management Service (handles both writes and the historical-query path) → DynamoDB (comments) + a Realtime Messaging layer that pushes new comments out to connected viewers.

## Deep dive 1 — real-time delivery mechanism

- **Bad — polling:** to feel real-time you'd need sub-second intervals at massive scale — wasteful and still laggy.
- **Good — WebSockets:** works, but is full bidirectional infrastructure for a traffic pattern that's overwhelmingly one-directional (thousands of viewers, comparatively few commenters) — a mismatch between the tool's cost and what's actually needed.
- **Great — SSE:** one-way server→client push over plain HTTP, which is the actual shape of this traffic — matches `systems-design-knowledge-base`'s networking guidance on choosing SSE specifically when the asymmetry is real.

## Deep dive 2 — scaling delivery to millions of viewers

- **Good — naive pub/sub:** every server subscribes to every video's channel and filters — wastes enormous compute broadcasting comments to servers with no interested viewers connected.
- **Great — partitioned pub/sub with viewer co-location:** consistent-hash `liveVideoId` (`hash(liveVideoId) % N`) combined with L7 load balancing so viewers of the same stream land on the same server, shrinking the number of channel subscriptions any one server needs.
- **Great — dispatcher service (alternative):** a centralized service tracks which servers hold which video's viewers and routes directly, avoiding pub/sub subscription-management complexity altogether. Presented as a genuine alternative, not strictly inferior to the consistent-hashing approach — the choice is an infrastructure-ownership tradeoff, not a correctness one.

## Deep dive 3 — mega-streams (100k+ concurrent viewers)

- **Good — sampling:** show a representative subset of comments rather than all of them, adjusting the sample rate to the actual comment velocity (e.g. showing only 1-2% of comments at 5,000/sec).
- **Great — CDN snapshots:** maintain a ring buffer of the ~100-200 most recent comments, snapshot it to Redis/CDN roughly every second; clients on a mega-stream poll the CDN snapshot instead of holding an SSE connection at all. Trades 1-2 seconds of extra latency for essentially unlimited scalability, by leaning on CDN infrastructure that already exists rather than building bespoke fan-out capacity for the rare mega-stream case.

## Deep dive 4 — reconnection / missed comments

SSE's own `Last-Event-ID` mechanism: on reconnect, the server replays whatever the client missed, bounded by a configurable replay window (e.g. the last 5 minutes) so a long-disconnected client doesn't trigger an unbounded replay.

## Deep dive 5 — pagination for historical comments

- **Bad — offset pagination:** the DB has to count all preceding rows on every page, and results shift under you if comments are added/removed while scrolling.
- **Great — cursor pagination:** use the comment ID as the cursor, backed by an index on that field — stable under concurrent inserts and cheap regardless of how deep the user has scrolled. DynamoDB supports this natively via `KeyConditionExpression`.

## Final architecture

Comment Management Service (writes + historical reads, DynamoDB) + Realtime Messaging layer (SSE, partitioned pub/sub or dispatcher-routed) + Redis/CDN (recent-comment cache for mega-streams and general snapshotting).

## Level expectations

- **Mid (80% breadth / 20% depth):** recognizes polling's limits, proposes a push-based model with some form of pub/sub, accepts interviewer guidance on the harder scaling questions.
- **Senior (60% breadth / 40% depth):** arrives at the pub/sub design independently, proactively raises scaling and its tradeoffs, articulates architectural decisions clearly rather than asserting them.
- **Staff+ (40% breadth / 60% depth):** anticipates the mega-stream problem and CDN-snapshot-style solutions unprompted, draws on real practical experience, minimal interviewer steering needed.
