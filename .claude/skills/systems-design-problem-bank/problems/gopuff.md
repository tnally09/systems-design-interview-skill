# Gopuff — Local/Rapid Delivery Inventory

Source: hellointerview.com problem-breakdowns/gopuff (free).

## Requirements

**Functional:** a user queries item availability by location, aggregated across nearby distribution centers (DCs) that can deliver within 1 hour; a user can place a multi-item order without any item being double-sold.
**Out of scope:** payments, driver routing, search/catalog browsing APIs, cancellations.

**Non-functional:** availability queries < 100ms; strong consistency for orders (no simultaneous double-purchase of the same physical unit); scale to 10k DCs, 100k distinct items, ~10M orders/day.
**Out of scope:** privacy, security, disaster recovery.

## Core entities

Inventory (a physical item instance at a specific DC), Item (the product type — e.g. "Cheetos"), DistributionCenter, Order (a set of ordered inventory + shipping/billing).

## API

Two endpoints: an availability query by location (paginated), and order placement (which also carries the user's location, since delivery eligibility depends on it).

## High-level design

**Availability flow:** find DCs within delivery radius of the user → query inventory across just those DCs → sum quantities per item → return. Served via an Availability Service backed by Postgres read replicas, with inventory data partitioned regionally.

**Order flow:** a single atomic Postgres transaction checks inventory for every item in the order, records the order, and updates inventory status — and fails the *entire* order if any single item is unavailable, rather than attempting a partial fulfillment. The source explicitly chose this single-transaction approach **over** a distributed-lock scheme specifically to avoid distributed-lock failure modes (orphaned locks, deadlocks across services) — directly the same lesson as Shopify's real production case in `systems-design-knowledge-base/reference/in-the-wild.md`: don't reach for a coordination layer when the data already lives in one database that can just do the transaction.

## Deep dive 1 — travel-time-aware DC selection

- **Bad:** simple Haversine (straight-line) distance — ignores actual roads/traffic, so "nearby" isn't really "deliverable in an hour."
- **Bad (the naive fix):** query a real travel-time service against every DC — far too many external API calls per request.
- **Great:** prune to a coarse radius first (e.g. 60 miles via Haversine) to cut the candidate set down cheaply, and only call the real travel-time service for that small, already-filtered set. The same two-phase "cheap prune, then expensive precise check" shape as proximity search generally (see `systems-design-knowledge-base/reference/deep-dives.md`).

## Deep dive 2 — availability query scalability

The source does the capacity math explicitly here because the caching decision genuinely turns on it: `10M orders/day ÷ 100k sec/day × 10 (browse:order ratio) ÷ 0.05 (order rate) ≈ 20k queries/sec`.
- **Great — Redis caching:** cache per-location/item availability with a 1-minute TTL; the Orders Service explicitly invalidates affected cache entries on a successful order rather than waiting out the TTL, so a just-sold-out item doesn't keep showing as available for up to a minute.
- **Great — Postgres read replicas + regional partitioning (by zipcode):** spreads read load across replicas and keeps a region's queries local to that region's partition; the real cost is replica sizing and rebalancing overhead as regional demand shifts over time. Presented as a complement to caching, not a replacement for it.

## Final architecture

Three services — Availability, Orders, and a shared Nearby (DC-proximity) service. Data layer: regionally partitioned Postgres with read replicas for availability queries and a leader (SERIALIZABLE isolation) for order transactions; Redis cache for inventory lookups (invalidated on order); an external travel-time service queried only for pre-filtered candidates.

## Level expectations

- **Mid:** ~80% breadth / 20% depth; defines the API and data model, covers both the availability and order routes at a functional level; interviewer may need to drive the later, deeper stages.
- **Senior:** ~60% breadth / 40% depth; proactively optimizes the critical paths (the DC-pruning and caching deep dives), articulates the architectural tradeoffs made rather than just stating the final choice, anticipates bottlenecks before being asked.
- **Staff+:** ~40% breadth / 60% depth; demonstrates real "been there" production judgment, exceptional proactivity, goes deep on 2-3 areas with genuine insight the interviewer didn't already have.
