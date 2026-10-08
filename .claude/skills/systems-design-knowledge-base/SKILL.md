---
name: systems-design-knowledge-base
description: >
  Reference knowledge base of hellointerview.com's system design core concepts, technology deep dives, common patterns, and real-world case studies. Not typically invoked directly by a user — the systems-design-interview, systems-design-teacher, and systems-design-problem-bank skills read from this one to ground their questions, feedback, and explanations in hellointerview's actual methodology rather than generic distributed-systems lore. A user can also invoke it directly to ask "what does hellointerview say about X" (caching, sharding, CAP, Redis, Kafka, consistent hashing, proximity search, etc).
---

# System Design Knowledge Base

Source: [hellointerview.com](https://www.hellointerview.com/learn/system-design) — the "Core Concepts," "Deep Dives," and "In the Wild" sections, plus the pattern references embedded in their problem breakdowns. This is free-tier content, condensed and paraphrased into reference notes here, not reproduced verbatim. Their **individual pattern deep-dive pages, "Numbers to Know," "Database Indexing," and several additional problem breakdowns and tech deep-dives (Postgres, Flink, ZooKeeper, vector DBs, CDC) are paywalled** and not included — don't present this file as more complete than it is on those specific topics.

This file is an index. The actual content lives in `reference/` — read the specific file you need rather than trying to hold all of it at once:

| File | Covers |
|---|---|
| `reference/core-concepts.md` | Networking essentials, API design, data modeling, caching, sharding, consistent hashing, CAP theorem — the 7 free core-concept pages |
| `reference/deep-dives.md` | Redis, Elasticsearch, Kafka, API Gateway, Cassandra, DynamoDB, Proximity Search, Time-Series Databases — the 8 free technology deep-dives |
| `reference/patterns.md` | The 8 named patterns (scaling reads/writes, real-time updates, long-running tasks, contention, large blobs, multi-step processes, proximity) with real examples pulled from the problem breakdowns, since the dedicated pattern pages are paywalled |
| `reference/in-the-wild.md` | 5 real production case studies (Shopify, Discord, Slack, Figma, Spotify) — what companies actually did and the design lesson each teaches |

## How to use this

**Recognize before reciting.** The value of this knowledge base isn't naming the right technology — it's recognizing which mechanism actually fits the problem in front of the candidate and being able to explain the tradeoff of choosing it over the obvious alternative. A candidate who says "Kafka" because it's the well-known answer, without being able to say why an alternative (SQS, a database queue table, Redis streams) wouldn't do, is exhibiting exactly the "resume-driven design" hellointerview's problem breakdowns repeatedly flag as a negative signal — see the Bad/Good/Great tiers in [[systems-design-problem-bank]] problems for what that looks like concretely.

**Match depth to what the design decision needs.** Most of these reference files include an explicit "when NOT to use this" section (e.g., Elasticsearch is wrong for < 100k documents or write-heavy workloads; Cassandra is wrong when you need joins or strict consistency; a time-series DB is wrong for high-cardinality tags). A candidate reaching for a specialized system without being able to say why the boring default (Postgres, a plain index, a single cache) wouldn't have worked is a gap worth probing — this is the same discipline as the "don't shard prematurely" stance on capacity math in the core interview skill: establish the need before reaching for the tool.

**Used by the other three skills like this:**
- [[systems-design-interview]] (Interviewer Mode) reads the relevant `reference/patterns.md` entry when deciding what to probe in a deep dive, and reads `reference/deep-dives.md`/`core-concepts.md` to judge whether a candidate's justification for a technology choice is actually sound.
- [[systems-design-interview]] (Critic Mode) uses the same files to check whether "Technical Excellence" claims in a design hold up, and whether the candidate reached for the pattern that actually fits.
- systems-design-teacher uses these files as the actual material it teaches from, walking a candidate through *why* (e.g.) cache-aside is the default pattern rather than just telling them to use it.
- systems-design-problem-bank's problem files cross-reference these concepts/patterns rather than re-explaining them.
