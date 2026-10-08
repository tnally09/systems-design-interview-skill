---
name: systems-design-problem-bank
description: >
  A bank of 15 detailed system design problems sourced from hellointerview.com's free problem breakdowns, each with requirements, core entities, API design, a high-level design, per-topic Bad/Good/Great solution tiers, and explicit Mid-level/Senior/Staff+ expectations. Used by systems-design-interview and systems-design-teacher to select a problem and to calibrate feedback/hints against a real answer key, without replacing their ability to invent a fresh company-tailored or random prompt. A user can also invoke this directly to browse available problems or ask what a strong answer to one of them looks like.
---

# Problem Bank

Source: hellointerview.com's free "Problem Breakdowns" section (15 of their problems are free; the rest — Yelp, Instagram, Strava, Robinhood, Google Docs, Payment System, Online Chess, ChatGPT, Flash Sale, and others — are paywalled and not included here). Each file under `problems/` is a condensed, paraphrased answer key, not a verbatim reproduction.

## The 15 problems

| File | Prompt | Core hard problem(s) |
|---|---|---|
| `problems/ticketmaster.md` | Event ticket booking platform | No-double-booking under contention, scaling view traffic, low-latency full-text search |
| `problems/bitly.md` | URL shortener | Short-code uniqueness at scale, extreme read/write skew |
| `problems/dropbox.md` | File storage & sync | Large-file upload (chunking, resumability), sync fan-out, security |
| `problems/fb-news-feed.md` | Social feed | Fan-out on read vs. write at extreme follow/follower counts, hot-post read skew |
| `problems/top-k-videos.md` | Real-time top-K video view leaderboard | Extreme write throughput, windowed aggregation, approximate vs. exact counting |
| `problems/tinder.md` | Dating app matching | Consistency on mutual-swipe matching, low-latency geo+preference feed, avoiding re-shown profiles |
| `problems/leetcode.md` | Coding judge + live leaderboard | Secure/isolated code execution at scale, real-time leaderboard updates |
| `problems/uber.md` | Ride-sharing dispatch | Real-time driver location + proximity matching, avoiding duplicate driver assignment, demand spikes |
| `problems/whatsapp.md` | Messaging | Connection routing at billions of users, delivery guarantees, multi-device fan-out |
| `problems/distributed-rate-limiter.md` | Rate limiter (infra-style prompt) | Algorithm choice, shared distributed state, fail-open vs. fail-closed |
| `problems/youtube.md` | Video upload & streaming | Transcoding pipeline (DAG), resumable upload, adaptive bitrate streaming |
| `problems/web-crawler.md` | Web crawler for LLM training data | Fault tolerance at scale, politeness/robots.txt, dedup, crawl-trap avoidance |
| `problems/ad-click-aggregator.md` | Real-time ad click analytics | Streaming aggregation, exactly-once/idempotency, hot-ad shard skew |
| `problems/fb-live-comments.md` | Live video comments | Real-time fan-out at massive concurrent-viewer scale, mega-stream degradation |
| `problems/fb-post-search.md` | Full-text post search | Inverted index design, multi-keyword query cost, write-heavy like counts |
| `problems/gopuff.md` | Local/rapid delivery inventory | Cross-DC inventory aggregation under a latency bound, no-oversell consistency |

Every file ends with the source's stated **Mid-level / Senior / Staff+ expectations** — use these as the actual calibration for the verdict, not a generic sense of "good."

## How the other skills should use this

**Selecting a problem.** When Interviewer or Teacher mode needs a prompt, treat this bank as one source among the ways to pick a prompt — not a replacement for inventing a company-tailored or random scenario as those skills already describe. Reasonable ways to use it:
- The user names a company/domain that maps cleanly to a bank problem (e.g. "Uber" → `uber.md`, "a dating app" → `tinder.md`) — pulling the real hellointerview problem is usually better than inventing a look-alike, since it comes with a genuine calibration answer key.
- The user asks for a random scenario — you can draw from the bank or invent one; don't default to the bank every time or the practice gets repetitive and memorizable. Mix in invented prompts, especially on repeat sessions with the same user.
- The user explicitly asks for one of these 15 by name.

**Never expose the answer key to a candidate mid-attempt.** In Interviewer Mode and Teacher Mode alike, the candidate must not see this file's content while they're working the problem — that defeats the exercise. Interviewer Mode doesn't consult it live at all beyond picking the prompt (it isn't grading). Teacher Mode may draw on a bank file's Bad→Good→Great framing *after* the candidate has made a genuine attempt at that specific sub-problem, to teach the escalation — never hand over the "Great" answer before the candidate has tried.

**Critic Mode grading a bank problem.** If the pasted prompt matches a bank problem, you may read that problem's file to sanity-check whether the candidate's design and deep dives land near the tiers described — but grade what's actually in front of you, per systems-design-interview's Critic Mode rules; a candidate reaching a different reasonable "Great" solution than the one in the file is not a deduction, and the Mid/Senior/Staff+ expectations here are a calibration anchor, not a checklist to match line-for-line.

**Cross-reference [[systems-design-knowledge-base]]** rather than re-deriving explanations of a pattern or technology named in a problem file — e.g. when a file says "Redis distributed lock with TTL," pull the mechanism detail from `systems-design-knowledge-base/reference/deep-dives.md` if it's needed, rather than re-explaining Redis from scratch.
