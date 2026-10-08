# In the Wild: Real Production Case Studies

Hellointerview's 5 free "In the Wild" case studies — what real companies actually did in production, and the design lesson each is meant to teach. These are useful in Teacher mode especially, to show that hellointerview's "establish the need before reaching for the specialized tool" discipline isn't just interview theater — it's how these decisions actually get made in practice.

## Shopify: Inventory Reservations (avoid the coordination layer you don't need)

**Problem:** preventing overselling at checkout, with reservation state in Redis and permanent inventory in MySQL — two systems that couldn't commit atomically together, so a crash between the two writes could corrupt the ledger.

**What they did:** moved reservations *into* MySQL itself, using a one-row-per-unit reservation pool (~1,000 rows per item) so concurrent buyers use `SELECT ... FOR UPDATE SKIP LOCKED` to each claim a different row instead of contending on one shared counter. Reservation and ledger updates now commit in a single transaction. They also tuned isolation level (READ COMMITTED to avoid gap-locking) and primary key ordering to cut lock overhead.

**Lesson:** *"If you're reaching for Redis, Kafka, or a custom coordination layer for high-throughput mutual exclusion, your existing database might already be enough."* Atomicity boundaries should drive architecture — operations that must commit together belong in the same database, and clever data modeling (spreading contention across many rows) can out-scale a hot-row design without adding a new system at all. Directly relevant to the Ticketmaster/Tinder/Uber "dealing with contention" deep dives in `reference/patterns.md` — this is the production version of "a plain transaction can beat a distributed lock."

## Discord: Message Storage at Scale (a faster DB and less DB work are different problems)

**Problem:** trillions of messages across 177 Cassandra nodes. Popular channels concentrated reads onto the same 3 replica nodes (hot partitions), and JVM GC pauses plus manual "gossip dance" node maintenance made the cluster operationally painful.

**What they did:** two independent fixes. (1) Migrated Cassandra → ScyllaDB, a C++ reimplementation with a shard-per-core architecture that removes JVM GC pauses. (2) Added a request-coalescing layer (Rust services) in front of the database: consistent-hash routing on channel ID sends identical concurrent requests to the same service instance, which merges overlapping reads into a single database query instead of forwarding each one.

**Result:** 177 nodes → 72; p99 latency 40-125ms → 15ms.

**Lesson:** a faster database alone doesn't stop thousands of clients from requesting the same data simultaneously — that's solved by doing less work before it reaches the database, not by a bigger/faster backend. Two different problems (per-operation latency vs. total redundant load) need two different fixes.

## Slack: Job Queue (bound the buffer by disk, not by the fast layer's RAM)

**Problem:** a Redis-based job queue (1.4B jobs/day peak) where the burst buffer and the active dispatch workspace shared the same bounded RAM pool. When downstream slowdowns caused backlog, Redis hit its memory ceiling and *both* enqueue and dequeue locked up — a total outage instead of graceful degradation.

**What they did:** put Kafka in front of Redis as a durable, disk-bounded buffer, connected by two new services — `Kafkagate` (accepts enqueues over HTTP, writes synchronously to Kafka) and `JQRelay` (drains Kafka into Redis at a controlled, throttleable rate, using distributed locks for single ownership per topic, only advancing Kafka offsets once a job safely lands in Redis).

**Lesson:** *"Keep the backlog somewhere that can't run out of memory, and let the fast layer hold only what's in flight."* This was a deliberately minimal change — they didn't replace Redis, they added a layer that separates unbounded durable buffering from the fast active-dispatch path, so resource contention in one can't cascade into total failure of the other. A strong example of "the great solution isn't always the most novel one" — same shape as several problem breakdowns' escalation from a single shared resource to a two-tier buffer/dispatch split.

## Figma: Multiplayer Editing (architectural constraints can dissolve the hard algorithmic problem)

**Problem:** multiple designers editing the same file concurrently, needing eventual convergence, offline support, instant local feedback, and safe tree restructuring (moving an object between containers without duplicating it).

**What they did:** skipped both full Operational Transformation and general CRDTs. Instead: one central server process per open document that receives all edits and establishes their ordering; last-writer-wins conflict resolution *per property* (color, position, etc.) rather than per-object; the parent relationship modeled as just another property, making "move to a different container" an atomic, identity-preserving operation; fractional positioning (values between 0 and 1) for sibling order so insertions never require renumbering.

**Lesson:** accepting a central authority (one server per document) eliminated the hardest part of the general problem — distributed consensus on event ordering — because a real requirement (there's always a server involved anyway) made the fully-decentralized case unnecessary. The broader point for interviews: understand what the problem *actually* requires, not the textbook-general version of it; trading algorithmic complexity for a bit of infrastructure/architectural constraint is often the more elegant answer, and reaching for the textbook-general solution (full CRDT) when a narrower one fits is itself a form of over-engineering.

## Spotify: Point Queries on a Data Lake (extend, don't replace, working infrastructure)

**Problem:** exabytes of listening history live in cheap object storage as column-oriented Parquet, great for big analytical scans but terrible for a single point query ("what did user X listen to?") — finding one user's rows means walking metadata/row-group/page chains across tens of thousands of files, pulling multi-MB pages for a few hundred bytes of actual data. The standard fix (copy hot data into a key-value store like Bigtable/DynamoDB) means maintaining and rationing a second, expensive copy of the data.

**What they did:** built RAP (Random Access Parquet), an external index that never modifies the existing Parquet files: bucket by user ID with cached Bloom filters to prune ~90,000 candidate files down to ~12 relevant ones without opening any of them; a definitive (not probabilistic) key→row-number index for O(1) lookups instead of Parquet's built-in probabilistic indexes; new files written pre-sorted by key so one user's data lands on one small page instead of scattered across many; small values copied directly into the index itself so some lookups need zero storage reads.

**Lesson:** this adds only an index (gigabytes per petabyte of underlying data), not a second full copy — and requires no pipeline rewrite or migration. The general interview-relevant point: a good design often *extends* the infrastructure and pipeline that already exist and is already trusted, rather than proposing a parallel system — "organic" system design beats a wholesale rewrite when the constraint is real production infrastructure, not a greenfield exercise.
