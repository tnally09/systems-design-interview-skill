# Core Concepts

Condensed from hellointerview's 7 free "Core Concepts" pages. Each entry: the key mechanism, the interview application, and common mistakes to watch for (as Interviewer/Critic) or teach (as Teacher).

## Networking Essentials

**Layers that matter:** Transport (TCP vs UDP) and Application (HTTP, REST/GraphQL/gRPC, SSE, WebSockets, WebRTC).

- **TCP vs UDP:** TCP (reliable, ordered, overhead) is the default; UDP ("fast but unreliable") only for latency-sensitive/loss-tolerant cases (video, gaming, VoIP).
- **SSE vs WebSockets:** SSE is one-way server→client over plain HTTP — use it whenever the traffic is fundamentally asymmetric (many more pushes than client messages), e.g. live comment feeds, notifications. WebSockets are full bidirectional and needed only when the client genuinely sends high-frequency messages back (chat, multiplayer editing). Don't reach for WebSockets by default — it's the higher-overhead option and a common resume-driven pick when SSE (or even polling) would do.
- **gRPC:** internal service-to-service, performance-critical paths. Not the default for a client-facing API.
- **Load balancing:** Layer 4 (TCP/UDP, fast, needed for persistent connections like WebSockets) vs Layer 7 (HTTP-aware routing by path/header, more flexible, more CPU). Algorithms: round robin, least connections, IP hash.
- **Failure handling:** exponential backoff with jitter on retries (never naive fixed-interval retry — it synchronizes retries into thundering herds), idempotency keys for safe retries of non-idempotent operations, circuit breakers (closed/open/half-open) to stop hammering a failing dependency.
- **Common mistakes to flag:** using UDP without justifying the loss tolerance; adding WebSockets without justifying the bidirectional need; assuming the network is reliable; treating HTTPS as a substitute for server-side validation of the request body.

## API Design

- **Default to REST.** GraphQL only when genuinely heterogeneous clients need different shapes from the same data. gRPC for internal, performance-critical calls.
- **Hellointerview's explicit stance:** "most interviewers don't care about your API design being perfect." Sketch 4-5 endpoints against the core entities in about 5 minutes and move to harder architectural problems — don't over-invest here. See `2b` in the core interview skill: this is one of the areas where thoroughness is a false signal of quality.
- **Principles that do matter when probed:** model around resources (nouns) not actions/verbs; consistent naming/pagination/error shape across endpoints; statelessness; idempotency for anything that mutates (esp. payments/bookings — use an idempotency key); paginate with a size cap (offset pagination is fine for most cases, cursor-based for real-time/frequently-mutating collections); auth derives the current user from a token, never trust a user/account ID passed in the request body.

## Data Modeling

- **The bar is lower than a dedicated data-modeling interview.** A clear, functional schema tied to the actual access patterns is enough — don't grade toward textbook normalization for its own sake.
- **Choosing a database type** is about the access pattern, not familiarity:
  - Relational (Postgres/MySQL) — default for most problems; clear entity relationships, need joins, ACID matters.
  - Document (Mongo/Firestore) — flexible/evolving schema, deeply nested data. Rarely the right call in an interview since scope is usually fixed up front.
  - Key-value (Redis/DynamoDB) — caching, session state, exact-key lookups only, no joins.
  - Wide-column (Cassandra/HBase) — massive write volume, time-series, query-pattern-driven (denormalized) modeling.
  - Graph (Neo4j) — "almost never in interviews"; treating this as a default for anything "social" is a common mistake to flag — even Facebook models its social graph on MySQL.
- **Three things should drive every schema decision, explicitly stated:** data volume, access patterns (which flow directly from the API you already defined — "what will each endpoint query?"), and consistency requirements (strong for money/inventory/bookings, eventual tolerable for feeds/social).
- **Keys:** system-generated IDs as primary keys, not business data (emails, usernames) that can change.
- **Indexing:** tie every proposed index to an actual query from the API — "index on posts.user_id because GET /users/:id/posts needs it," not indexes proposed in the abstract.
- **Normalize first, denormalize deliberately.** Valid reasons to denormalize: analytics/reporting, audit logs/event snapshots, read-optimized systems where staleness is acceptable. Prefer keeping the source of truth normalized and denormalizing into a cache/read-model over denormalizing the primary tables outright.
- **Sharding tie-in:** shard key should match the dominant access pattern (e.g., shard by user_id if "get this user's data" dominates) to avoid cross-shard queries. Time-range shard keys create hot shards on the current partition — a common anti-pattern to catch.

## Sharding

- **Sharding ≠ partitioning.** Partitioning splits a table within one database instance; sharding spreads data across independent machines.
- **The core discipline hellointerview repeats everywhere sharding comes up: don't introduce it prematurely.** Establish, with actual numbers, that a single instance's storage or throughput is genuinely insufficient before proposing it. This is the same "capacity discipline" principle from the core interview skill applied specifically to this decision — sharding is the textbook example of "a scale-driven decision that needs grounding when asserted."
- **Good shard key:** high cardinality, evenly distributed, aligned with the dominant query (so most queries hit one shard). `user_id`/`order_id` — good. Booleans or `created_at` — bad (hot spots).
- **Distribution strategies:** range-based (simple, hot-spot prone), hash-based (even, but resharding is expensive without consistent hashing), directory-based (flexible, adds a lookup hop and a critical dependency).
- **Hard problems:** hot spots (the "celebrity" user/key — isolate to a dedicated shard or use a compound key), cross-shard queries (expensive — mitigate via caching/denormalization or design to avoid them by keeping a user's data co-located), cross-shard transactions (avoid needing them at all via shard-key design rather than trying to solve distributed transactions).
- **In practice:** most systems lean on a database that shards for you (DynamoDB, Cassandra, MongoDB) or a sharding layer (Vitess, Citus) rather than hand-rolling it — know when to say "the database handles this" vs. when the interview specifically wants the mechanism explained (infrastructure-heavy prompts).

## Consistent Hashing

- **Problem it solves:** naive `hash(key) % N` remaps almost everything when N (number of nodes) changes, causing a redistribution storm on every scale-up/down.
- **Mechanism:** nodes and keys placed on a hash ring (0 to 2^32-1); a key belongs to the first node clockwise from it. Adding/removing a node only remaps the data between it and its neighbor — a small fraction, not everything.
- **Virtual nodes:** each physical node gets multiple ring positions so that a failure/addition spreads load across many neighbors instead of dumping it all on one.
- **Consistent hashing tells you *where* data should live; it doesn't by itself provide durability** — real systems pair it with replication (DynamoDB across 3 AZs, Cassandra replicating to N consecutive nodes) so failures don't require immediate data movement.
- **Hot spots are a separate problem from structural imbalance.** Consistent hashing fixes distribution across nodes; it does *not* fix one key being disproportionately popular — that needs read replicas for the hot key, key-space salting, or adaptive rebalancing.
- **Interview calibration:** most systems (DynamoDB, Cassandra) already do this under the hood — usually it's enough to say so. Go deep only when the prompt is explicitly about building distributed infrastructure (a distributed cache, a distributed database, a message broker) — see `reference/deep-dives.md`.

## CAP Theorem

- **Partition tolerance is not optional in a distributed system** — the real choice interviews care about is consistency vs. availability *during a partition*.
- **Framing that should show up in non-functional requirements, not as a theorem recitation:** "would it be catastrophic if users briefly saw inconsistent data?" If yes (money, inventory, bookings, seat reservations) — choose consistency. If no (social feeds, profile views, content metadata) — choose availability.
- **This should drive concrete technology choices, not just a stated preference:** consistency-leaning → single-node/leader-based writes, distributed transactions, Postgres/Spanner-style systems. Availability-leaning → read replicas, async replication, Cassandra/DynamoDB-style systems.
- **The same system often needs both stances in different places** — e.g. Ticketmaster: strong consistency for seat booking, availability for browsing; Tinder: consistency for match confirmation, availability for profile viewing. A design that applies one blanket consistency stance to the whole system without distinguishing subsystems is under-thought — probe for this split explicitly.
