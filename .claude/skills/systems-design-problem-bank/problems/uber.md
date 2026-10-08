# Uber — Ride-Sharing Dispatch

Source: hellointerview.com problem-breakdowns/uber (free).

## Requirements

**Functional:** rider requests a fare estimate (pickup + destination → quote); rider confirms a ride from the estimate; system matches the rider with a nearby available driver; driver accepts/declines and navigates to pickup/dropoff.
**Out of scope:** post-trip ratings, advance scheduling, ride tiers/categories.

**Non-functional:** matching latency < 1 minute; strong consistency on assignment (no double-assigning a driver); 100k concurrent requests.
**Out of scope:** security/privacy compliance, general resilience/monitoring/CI-CD (explicitly descoped by the source, not implying they don't matter in reality).

## Core entities

Rider, Driver (vehicle + availability status), Fare (estimate, ETA, pickup/destination), Ride (confirmation through completion), Location (real-time driver coordinates + timestamp).

## API

```
POST  /fare                     { pickupLocation, destination } → Fare
POST  /rides                    { fareId } → Ride
POST  /drivers/location         { lat, long }
PATCH /rides/:rideId            { accept | deny } → Ride
```
Security note baked into the source's own API design: user identity always comes from the session/JWT, never a client-supplied field — and fare/timestamps are server-computed, never trusted from the client.

## High-level design (staged)

1. Fare estimate: Ride Service calls a third-party maps API, persists nothing yet beyond the quote.
2. Ride request: creates a Ride row, status `requested`.
3. Driver matching: a Location Service ingests periodic driver pings; a Ride Matching Service finds nearby available drivers.
4. Acceptance: a Notification Service pushes the request (APNs/FCM); driver responds via the PATCH endpoint.

## Deep dive 1 — driver location updates + proximity search

At 10M drivers pinging every 5s, that's ~2M writes/sec, and naive lat/long `WHERE` scans are a non-starter.
- **Bad:** direct DB writes + naive proximity queries — full scans, overload.
- **Good:** batch location writes + a geospatial-aware store (PostGIS, quad-trees) — cuts write pressure but introduces staleness.
- **Great:** Redis geospatial commands (`GEOADD`/`GEOSEARCH`, geohash-backed) for real-time driver positions, with TTL-based auto-expiry (~30s) so offline drivers silently drop out of matching. Durability risk (in-memory) is mitigated with RDB/AOF persistence plus Sentinel failover — see `systems-design-knowledge-base/reference/deep-dives.md`'s Redis entry.

## Deep dive 2 — the ping volume itself is the load, not just the query

**Great:** adaptive client-side ping frequency — the driver app adjusts how often it sends location based on speed, direction change, and proximity to a pending match, rather than a fixed interval. Explicitly framed by the source as "don't neglect the client" — the fix isn't purely server-side.

## Deep dive 3 — preventing duplicate driver assignment

- **Bad:** an application-level lock with no shared coordination — races under multiple Ride Matching Service instances, no recovery story if a service crashes mid-hold.
- **Good:** a DB status field (`outstanding_request`/`accepted`/`available`) with an in-memory timeout — loses that timeout state if the service restarts.
- **Great:** a Redis distributed lock with a TTL matching the acceptance window (10s). Auto-expiry means a non-responding driver's lock releases itself and the next driver can be tried, with no manual cleanup logic needed. Requires Redis HA (Sentinel), but the short TTL keeps the blast radius of a Redis hiccup small.

## Deep dive 4 — dropped requests during demand spikes

- **Bad:** pure first-come-first-served with no buffering — a demand spike drops requests outright, and a service crash loses in-flight ones.
- **Great:** a Kafka queue absorbing ride requests, geographically partitioned, with dynamic consumer scaling; offsets committed only after a successful match, so a crashed consumer resumes cleanly; a priority queue variant avoids FIFO head-of-line blocking for e.g. surge-priority requests.

## Deep dive 5 — non-responsive drivers

- **Good:** a delay queue (e.g. SQS delay) retries the next-nearest driver after a timeout; gets messy with cascading delayed messages needing cancellation when a match succeeds elsewhere.
- **Great:** a durable execution framework (Temporal / AWS Step Functions) — the retry/timeout/fallback-to-next-driver logic is expressed as a workflow that survives service crashes, instead of hand-rolled delay-queue bookkeeping. Real cost to name: the team now owns learning/operating a new orchestration system.

## Deep dive 6 — overall scaling and latency

**Great:** geographic sharding — services, queues, and data partitioned by region, reducing client-server distance; consistent hashing for even distribution; cross-region ("scatter-gather") queries reserved for genuine boundary cases (e.g. a rider near a region edge), not the common path.

## Final architecture

Clients (adaptive-ping driver app) → API Gateway → Ride Service (fares) + Location Service (Redis geo) + Ride Matching Service (Redis locks, durable execution) + Notification Service → Kafka (request buffering) + relational DB (rides/users) → geographically sharded deployment.

## Level expectations

- **Mid (E4):** clean API/data model, functional end-to-end design, recognizes the need for spatial indexing without necessarily landing the specific mechanism, implements at least the "Good" tier for locking.
- **Senior (E5):** speeds through the basics to spend real time on 2+ deep dives, states tradeoffs explicitly, proactively surfaces bottlenecks rather than waiting to be pointed at them.
- **Staff+ (L6+):** ~40% breadth / 60% depth; anticipates problems and proposes solutions unprompted, draws on real production-scale experience, the interviewer should come away having learned something.
