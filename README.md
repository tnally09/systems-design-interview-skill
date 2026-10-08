# Systems Design Interview Skill

A set of four Claude Code skills for systems design interview practice, built around [hellointerview.com](https://www.hellointerview.com/learn/system-design/in-a-hurry/introduction)'s methodology rather than generic distributed-systems interview folklore.

## The four skills

- **`systems-design-interview`** — the core practice loop, run as two separate modes in two separate chat sessions:
  - **Interviewer mode** conducts a live mock interview: picks or scopes a design prompt, drives the session through hellointerview's delivery framework (requirements, core entities, API, high-level design, deep dives, failure modes/tradeoffs), and chases vague answers before letting the candidate move on. Ends with a neutral, non-editorializing writeup of the final design.
  - **Critic mode** grades a finished design writeup cold, with no visibility into how the interview session went, so scoring isn't biased by effort or back-and-forth. Grades against hellointerview's four-competency rubric (Problem Navigation, Solution Design, Technical Excellence, Communication & Collaboration) and produces a calibrated Strong Hire → Strong No Hire verdict.

  The two modes are kept blind to each other by design: the Critic only ever sees the final design output, never the interview transcript.

  Grading follows hellointerview's stance where it diverges from generic interview-prep advice — most notably: back-of-envelope math is not a required phase. It's treated as a tool to reach for only when a specific decision (e.g. sharding, adding a distributed cache) actually turns on the number, not a box to check up front.

- **`systems-design-teacher`** — a third mode for learning rather than evaluation: the same delivery-framework structure, but hands-on — concepts explained proactively, hints offered readily, no grading or verdict ever. Use this instead of Interviewer mode when the goal is understanding, not signal.

- **`systems-design-knowledge-base`** — reference notes distilled from hellointerview's free Core Concepts (7), Technology Deep Dives (8: Redis, Elasticsearch, Kafka, API Gateway, Cassandra, DynamoDB, Proximity Search, Time-Series DBs), Patterns (8 named patterns), and In-the-Wild real production case studies (5: Shopify, Discord, Slack, Figma, Spotify). The other three skills read from this rather than reasoning about technology from general knowledge; it's also invocable directly to ask "what does hellointerview say about X."

- **`systems-design-problem-bank`** — 15 fully worked problems from hellointerview's free Problem Breakdowns (Ticketmaster, Bitly, Dropbox, FB News Feed, YouTube Top-K, Tinder, LeetCode, Uber, WhatsApp, Distributed Rate Limiter, YouTube, Web Crawler, Ad Click Aggregator, FB Live Comments, FB Post Search, Gopuff), each with requirements, entities, API, high-level design, Bad/Good/Great solution tiers per deep dive, and explicit Mid-level/Senior/Staff+ expectations. Used by Interviewer/Teacher mode to select a prompt (without replacing their ability to invent a fresh company-tailored or random one) and by Critic mode as a calibration anchor.

**Paywall note:** hellointerview locks a chunk of their content — individual pattern deep-dive pages, "Numbers to Know," "Database Indexing," several additional problem breakdowns (Yelp, Instagram, Strava, Robinhood, Google Docs, and others), and a few extra tech deep-dives (Postgres, Flink, ZooKeeper, vector DBs, CDC). None of that is reproduced here; `systems-design-knowledge-base` and `systems-design-problem-bank` are built entirely from their free content.

## Usage

Invoke whichever skill fits in Claude Code:

```
/systems-design-interview
/systems-design-teacher
/systems-design-knowledge-base
/systems-design-problem-bank
```

For `systems-design-interview` or `systems-design-teacher`, either name a company/domain to tailor the prompt to, ask for a random scenario, or name one of the problem bank's 15 problems directly.

## Contents

- `.claude/skills/systems-design-interview/SKILL.md`
- `.claude/skills/systems-design-teacher/SKILL.md`
- `.claude/skills/systems-design-knowledge-base/SKILL.md` + `reference/*.md`
- `.claude/skills/systems-design-problem-bank/SKILL.md` + `problems/*.md`
