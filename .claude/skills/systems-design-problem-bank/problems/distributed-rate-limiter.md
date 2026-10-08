# Distributed Rate Limiter

Source: hellointerview.com problem-breakdowns/distributed-rate-limiter (free). An infrastructure-style prompt rather than a product — the "requirements" are closer to a spec than a scoping negotiation.

## Requirements

**Functional:** identify a client by user ID, IP, or API key; enforce configurable rules (e.g. 100 req/min); reject over-limit requests with HTTP 429 plus informative headers.
**Non-functional:** < 10ms added latency per request; high availability, eventual consistency acceptable; 1M req/sec across 100M DAU.

## Core entities

Rule (a limiting policy), Client (the thing being limited — user/IP/API key — with associated counter state), Request.

## System interface

```
isRequestAllowed(clientId, ruleId) → { passes: boolean, remaining: number, resetTime: timestamp }
```

## High-level design — where to place it

- **Bad — in-process:** each server keeps its own local counter, no global visibility; trivially bypassed by spreading requests across servers.
- **Good — dedicated rate-limiting microservice:** centralized global state, but adds a network hop (and a new failure point) to every single request.
- **Great — at the API Gateway/load balancer:** enforced at the system edge before any application server sees the traffic; the tradeoff is that it only has whatever context is present in the raw HTTP request (headers, IP) to identify the client.

## Algorithm choice

- **Fixed window counter:** simple, but allows a 2x burst at window boundaries (max requests at the end of one window plus the max again at the start of the next).
- **Sliding window log:** exact, but stores every timestamp per client — memory-expensive at real scale.
- **Sliding window counter:** a fixed-window hybrid that weights the previous window's count — good approximation, low memory, assumes roughly even traffic distribution.
- **Token bucket (recommended default):** a bucket refills at a steady rate and each request consumes a token; naturally handles both sustained load and legitimate bursts. State per client is just `(tokens, last_refill_time)`.

## Shared state & correctness

Redis holds the counters/buckets so every gateway instance sees the same state. **Race condition:** concurrent requests reading-then-writing a counter can double-count — the fix is a Lua script doing the read-check-update atomically in one round trip, not separate GET/SET calls.

429 responses should carry `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, and `Retry-After` — this is part of what the source treats as baseline-correct, not a stretch goal.

## Deep dive 1 — scaling to 1M req/sec

A single Redis instance tops out around 50-100k rate-limit checks/sec. **Fix:** shard Redis by consistent-hashing the client identifier (so each client's state lives on exactly one shard), or lean on Redis Cluster's built-in hash-slot distribution instead of hand-rolling the consistent-hashing logic. ~10 shards × ~100k ops/sec each reaches the target.

## Deep dive 2 — availability when Redis is down

This is a genuine, stated tradeoff, not a "great" answer that dominates:
- **Fail-closed** (reject everything): safe, but takes the whole API offline during a Redis outage — appropriate when unmetered traffic during an outage risks cascading failure downstream.
- **Fail-open** (let everything through): keeps the API up, but removes protection exactly when a Redis outage may coincide with a traffic spike — the worst time to lose it.
Mitigate either way with Redis master-replica replication and automatic failover (Redis Cluster's built-in failure detection, or Sentinel).

## Deep dive 3 — latency minimization

Connection pooling to Redis (avoid a TCP handshake per request); geographically distributed rate-limiting infrastructure close to users, at the explicit cost of added cross-region consistency complexity if state needs to be shared globally.

## Deep dive 4 — hot keys (one client generating extreme volume)

Distinguish legitimate from abusive: a legitimate high-volume client gets client-side rate limiting / request batching / a premium tier with dedicated infrastructure; an abusive one gets automatic escalating blocks or gets handed off to a dedicated DDoS-protection layer (Cloudflare/AWS Shield) rather than solved inside this system at all.

## Deep dive 5 — updating rules without a redeploy

- **Poll-based** (the common case): gateways re-read a config store every ~30s — simple, acceptable delay for most cases.
- **Push-based** (ZooKeeper/Redis pub-sub): near-instant propagation, more operational complexity — justify this only for genuinely urgent cases (security incidents, trading-style systems), not as a default.

## Final architecture

API Gateways (token-bucket check via Lua script) → sharded Redis Cluster (replicated, failover-capable) ← a config store feeding rule updates (poll or push).

## Level expectations

- **Mid:** explains one algorithm (token bucket is the expected default), places the limiter at the gateway, knows to use Redis for shared state, recognizes sharding will eventually be needed.
- **Senior:** real tradeoff depth on placement and algorithm choice, comfortable with consistent hashing/Redis Cluster/connection pooling, proactively raises hot keys and fail-open-vs-closed with a stated opinion.
- **Staff+:** speaks from production experience at comparable scale, proactively raises multi-region deployment, observability, and rollout/canary procedures unprompted.
