# LeetCode — Coding Judge & Live Leaderboard

Source: hellointerview.com problem-breakdowns/leetcode (free).

## Requirements

**Functional:** browse a paginated problem list; view a specific problem with language-specific starter code; submit a solution and get feedback; view a live competition leaderboard.
**Out of scope:** auth, profiles, payments, analytics, social features.

**Non-functional:** availability over consistency; strict isolation/security for arbitrary user code execution; submission results within 5s; support 100k concurrent competition users.

## Core entities

Problem (statement, test cases, expected outputs, per-language stubs), Submission (user code + results), Leaderboard (competition rankings).

## API

```
GET  /problems?page=&limit=
GET  /problems/:id?language=
POST /problems/:id/submit → Submission
GET  /leaderboard/:competitionId?page=
GET  /check/:submissionId
```

## High-level design

Monolithic API server + NoSQL store (DynamoDB) for problems/submissions + a container runtime for isolated execution + Redis for leaderboard performance + a queue buffering submissions under load.

## Deep dive 1 — safely executing untrusted code

- **Bad:** run submitted code directly in the API process — trivially exploitable (data deletion, crypto-mining, DDoS launch pad).
- **Good:** full VMs per submission — isolated, but heavy resource cost and slow cold-start makes it a poor fit for a 5-second budget.
- **Great:** Docker containers — much faster startup, still real isolation, hardened with: read-only filesystem, hard CPU/memory limits, an explicit 5-second execution timeout, no network access, syscall filtering via seccomp.

## Deep dive 2 — real-time leaderboard under competition load

- **Bad:** clients poll the database directly every few seconds — millions of redundant queries during a live competition.
- **Good:** periodic (e.g. every 30s) cache refresh — cuts DB load but sacrifices freshness during a fast-moving competition.
- **Great:** Redis sorted sets (`ZADD competition:leaderboard:{id} {score} {userId}`) updated in real time as submissions are judged; clients still just poll the cache (e.g. every 5s), but the expensive ranking computation itself is O(log N) in Redis instead of a repeated DB aggregation.

## Deep dive 3 — scaling to 100k concurrent competition users

Back-of-envelope the naive case explicitly: 10k simultaneous submissions × ~100 test cases each, judged within a minute, needs on the order of 1,600+ CPU cores if run serially per submission — vertical scaling alone is a non-starter.
- **Great:** horizontal auto-scaling of the container fleet keyed on CPU utilization, plus a queue (SQS) between the API and the execution fleet to absorb bursts, enable retries on transient failure, and make the whole flow asynchronous (client polls `GET /check/:id` rather than holding a request open).

## Deep dive 4 — test-case execution mechanics

Serialize test inputs/outputs to a language-agnostic format (e.g., arrays for a tree via level-order traversal); each supported language gets a deserialization + harness layer that reconstructs the language-native object, calls the user's function, and diffs the output. Worth naming explicitly if the interviewer pushes on "how does one judge actually work across 10 languages" — it's easy to hand-wave.

## Final architecture

API Server (stateless) → DynamoDB (problems, submissions) + SQS (submission buffering) → auto-scaled Docker execution fleet (hardened, isolated) → Redis (leaderboard) ← clients poll for both submission status and leaderboard.

## Level expectations

- **Mid (IC4):** clean API/data model; proposes some form of isolation (container/VM/serverless) without necessarily justifying the choice; grasps the high-level shape.
- **Senior (IC5):** deep-dives code-execution security specifically, justifies container vs. VM vs. serverless with real tradeoffs, discusses the actual test-harness implementation, proactively surfaces the throughput bottleneck.
- **Staff+:** drives the whole conversation, identifies design issues before being asked, actively minimizes complexity while keeping a credible scaling path.
