# Ticketmaster — Event Ticket Booking Platform

Source: hellointerview.com problem-breakdowns/ticketmaster (free).

## Requirements

**Functional (top 3):** view events; search events; book tickets to an event.
**Out of scope:** viewing a user's own booked events, admin event creation, dynamic pricing.

**Non-functional:** availability for search/view, but strong consistency for bookings (no double-booking); handle a single event selling to ~10M interested users; search latency < 500ms; read-heavy overall (~100:1 read/write).

## Core entities

Event, User, Performer, Venue, Ticket (one row per seat: event, seat, price, status), Booking (groups tickets purchased together).

## API

```
GET  /events/:eventId → Event & Venue & Performer & Ticket[]
GET  /events/search?keyword=&start=&end=&pageSize=&page= → Event[]
POST /bookings/:eventId  { ticketIds: string[], paymentDetails } → bookingId
```

## High-level design

Simple version: view/search hit the events DB directly; booking is a single Postgres transaction that checks ticket availability, flips status, and creates the booking row (ACID). **Problem this immediately exposes:** a user can select seats, stall on the payment form, and lock those seats out from everyone else indefinitely.

## Deep dive 1 — no-double-booking under contention

- **Bad:** hold a `SELECT FOR UPDATE` lock for the whole checkout (minutes). Kills throughput, risks deadlocks, doesn't survive a crashed client.
- **Good:** ticket status = available/reserved/booked, reservation carries an expiration timestamp, a cron job sweeps expired reservations back to available. Works, but there's a window where an expired-but-uncronned seat looks unavailable, and a cron failure can strand seats.
- **Great (chosen):** two variants, either defensible —
  1. *Implicit status:* a ticket is "available" if its status is available OR its reservation timestamp has passed — no cron needed, checked inline on every short transaction.
  2. *Redis distributed lock:* `SET ticketId userId NX EX 600` acquires a 10-minute reservation; releases automatically via TTL if payment never completes, or is released manually on success/cancel. The DB itself only ever has two ticket states (available/booked) — Redis carries the transient "reserved" state. If Redis fails, the DB's own optimistic check still prevents a double-sell, it just degrades UX. The seat-map read path (showing reserved seats as taken) uses a Redis sorted set scored by expiration so stale reservations don't need active cleanup.

**Full booking flow (Great):** acquire Redis lock → create in-progress booking row → user pays via Stripe (client-side tokenized) → Stripe webhook confirms → webhook idempotently (by bookingId) flips ticket→sold and booking→confirmed.

## Deep dive 2 — scaling view traffic for on-sale spikes

Cache event/venue/performer detail (read-through, Redis/Memcached, TTL tuned to how static the field is), horizontally scale a stateless read service behind a load balancer. Straightforward — the interesting content is deep dive 1 and 3, not this one.

## Deep dive 3 — UX under extreme demand (mega on-sales)

- **Good:** SSE pushes live seat-map updates so users aren't polling — but for something like a stadium show, seats still vanish faster than a user can act.
- **Great:** an admin-enabled virtual waiting queue gated in front of the booking page for flagged high-demand events. Queue = Redis sorted set ordered by join timestamp; users get an SSE/WebSocket connection showing live position/ETA; a controlled dequeue rate admits users (marked in a `admitted:{eventId}` set with TTL) before the Booking Service will accept their reservation attempt.

## Deep dive 4 — low-latency search

- **Good:** proper indexes on name/date/performer/venue, query hygiene (avoid `SELECT *`, avoid leading-wildcard `LIKE`).
- **Great option A:** Postgres full-text (`tsvector` + GIN index) or MySQL native full-text — much better than `LIKE` for substring matches.
- **Great option B:** Elasticsearch — inverted index, fuzzy matching (typo tolerance), synced from Postgres via CDC. Chosen when fuzzy/relevance matters more than plain substring speed.

## Deep dive 5 — speeding up repeated searches

- **Good:** cache full search-result pages in Redis, keyed by the full parameter set, short TTL.
- **Great:** lean on Elasticsearch's own shard-level query/request caching plus CDN edge caching for the (common) case where search results aren't personalized.

## Final architecture

Client → API Gateway → {Event Service, Search Service, Booking Service} (stateless, horizontally scaled) → Postgres (bookings, ACID) + Redis (reservation locks, waiting queue, caching) + Elasticsearch (search, CDC-synced) → Stripe (payment, webhook-driven) → CDN (search/edge caching).

## Level expectations (source's own framing)

- **Mid (E4):** working shortening/booking flow; recognizes the double-booking risk and lands the "Good" status+cron fix; basic indexing; needs prompting toward caching.
- **Senior (E5):** proactively reaches the "Great" contention fix and justifies it over the DB-lock alternative; uses Elasticsearch for search with a stated reason; discusses sharding/replication scaling; ~60% breadth / 40% depth.
- **Staff+:** breezes the basics, drives 2-3 deep dives unprompted (contention, mega-event UX, search), discusses security/product edges (predictable IDs, custom-alias collisions) and operational concerns; ~40% breadth / 60% depth.
