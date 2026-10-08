# Tinder — Dating App Matching

Source: hellointerview.com problem-breakdowns/tinder (free).

## Requirements

**Functional:** create a profile with preferences (age range, interests, max distance); view a stack of candidate profiles matching preferences+location; swipe right/left; get notified on a mutual (match) right-swipe.
**Out of scope:** photo upload, DMs, premium features.

**Non-functional:** strong consistency specifically for swipe/match recording; scale to 20M DAU × ~100 swipes/day; feed load < 300ms; never re-show a profile the user already swiped on.

## Core entities

User (profile + preferences + location), Swipe (swiping_user → target_user, yes/no), Match (created on mutual right-swipes).

## API

```
POST /profile                      create/update preferences
GET  /feed?lat=&long=              candidate profile stack
POST /swipe/{userId}               submit a swipe decision
```

## High-level design

Profile creation is a plain CRUD write. Feed generation is a filtered geo+preference query. Swiping writes to a dedicated Swipe Service/DB (Cassandra, chosen for write throughput) and checks for the inverse swipe in the same operation. Matches trigger a push notification (APNs/FCM) to the original swiper.

## Deep dive 1 — consistency on swiping/matching

- **Bad:** polling for a match. High latency, doesn't actually notify anyone promptly.
- **Good:** database transactions — but Cassandra's lightweight transactions (LWT) only work within a single partition, and 2B swipes/day across arbitrary user pairs won't fit that constraint as a blanket approach.
- **Great option A — engineered partition key:** key the Swipe/Match table by a canonical pair, e.g. `min(id1,id2):max(id1,id2)`, so both directions of a mutual swipe always land in the same partition and can use a single-partition atomic operation. Tradeoff: partitions for very active users grow unbounded over time and need a cleanup/archival strategy.
- **Great option B — Redis for the atomic check:** a Lua script records the swipe and checks-for-match atomically in Redis; Cassandra is the durable archive underneath. Tradeoff: Redis memory management, offset by being able to expire recent-swipe data aggressively since Cassandra already has it durably.

## Deep dive 2 — low-latency, geo-filtered feed generation

- **Bad:** a direct DB query with geospatial `WHERE` filters — doesn't scale.
- **Good option 1 — Elasticsearch with geo indexing:** fast, flexible multi-attribute filtering, but needs CDC sync from the primary DB, introducing eventual-consistency risk on preference/location changes.
- **Good option 2 — precompute + cache the feed:** cheap reads, but users burn through a cached feed quickly and stale entries appear if preferences/location change mid-cache-life.
- **Great — hybrid:** serve the cache first, fall through to a live Elasticsearch query once a user has exhausted it or gone stale; background refresh on a short (<1h) TTL, with refresh triggered specifically by detected preference/location changes for recently-active users rather than blindly for everyone.

## Deep dive 3 — never re-showing a swiped profile

- **Bad:** query the swipe history table and filter in the app layer — slow once a user has a long history, and can be inconsistent under the system's availability-leaning stance.
- **Good:** a client-side cache of the user's last-K swipes plus a DB filter — still slow for long histories.
- **Great:** a Bloom filter per user once their swipe history passes a size threshold. Tradeoff to name explicitly: false positives are possible (a never-swiped profile occasionally hidden — acceptable, since it just means one fewer candidate shown, not a correctness bug), false negatives are not (a swiped profile never mistakenly re-shown); rebuilding a user's Bloom filter from raw swipe data is itself an expensive operation to manage at scale.

## Final architecture

API Gateway → Profile/Swipe Services → Cassandra (durable swipe/match store, engineered partition keys) + Elasticsearch (geo-indexed feed, CDC-synced) + Redis (atomic match detection, feed caching) → APNs/FCM (match notifications).

## Level expectations

- **Mid (E4):** clean API/data model, functional feed/swipe/match design, handles geo filtering and the no-re-show requirement at a surface level; interviewer probes and partially drives.
- **Senior (E5):** ~60% breadth / 40% depth; proactively raises feed staleness, discusses index-type tradeoffs, details the consistency mechanism rather than asserting "it's consistent."
- **Staff+ (C6+):** ~40% breadth / 60% depth; production-grade proactivity, deep distributed-systems fluency under load, peer-level discussion quality.
