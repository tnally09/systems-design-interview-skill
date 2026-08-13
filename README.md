# Systems Design Interview Skill

A Claude Code skill for mock systems design interview practice, run as two separate modes in two separate chat sessions:

- **Interviewer mode** conducts a live mock interview: picks or scopes a design prompt, drives the session like a real onsite (requirements, capacity estimation, architecture, data model, deep dives, failure modes, tradeoffs), and chases vague answers before letting the candidate move on. Ends with a neutral, non-editorializing writeup of the final design.
- **Critic mode** grades a finished design writeup cold, with no visibility into how the interview session went, so scoring isn't biased by effort or back-and-forth. Produces a calibrated Strong Hire → Strong No Hire verdict.

The two modes are kept blind to each other by design: the Critic only ever sees the final design output, never the interview transcript.

## Usage

Invoke the skill in Claude Code:

```
/systems-design-interview
```

Then either name a company/domain to tailor the prompt to, or ask for a random scenario.

## Contents

- `.claude/skills/systems-design-interview/SKILL.md` — the skill definition.
