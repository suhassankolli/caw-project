---
name: adr-writer
description: Write an Architecture Decision Record to docs/adr/{yymmdd}-{title}.md when the user states a technical decision and wants it recorded as an ADR.
---

# ADR Writer

Turn a decision the user states (for example "use Redis Streams instead of Kafka for async jobs because Kafka is too heavy to operate") into an Architecture Decision Record.

## Steps

1. **Parse the decision.** Identify what was chosen, what it replaces or beats, and the stated reason.
2. **Gather context.**
   - Read existing docs in the repo (e.g. docs/design/, earlier ADRs in docs/adr/, any open decisions register) for related constraints and terminology.
   - Do your own research on the options (trade-offs, operational cost, scaling limits, maturity) using web search where useful. Cite sources at the end when you used them.
3. **Ask when unclear.** If the scope, the alternatives, the status, or key constraints (scale, team skills, budget, deadlines) are ambiguous and would change the record, ask the user short questions before writing. Do not invent facts about their system.
4. **Name the file.** `docs/adr/{yymmdd}-{title}.md`, where yymmdd is today's date and title is a short kebab-case slug of the decision (e.g. `261120-use-redis-streams-for-async-jobs.md`). Create docs/adr/ if missing. Do not overwrite an existing file; add a suffix instead.
5. **Write the ADR** with this structure:

```markdown
# {Title}

## Status
Proposed | Accepted | Deprecated | Superseded by [ADR](link)
Date: YYYY-MM-DD

## Context
The problem, forces and constraints that make a decision necessary.

## Decision
What we will do, stated plainly in active voice ("We will use...").

## Consequences
### Positive
- ...
### Negative / risks
- ...
### Follow-up actions
- ...

## Alternatives considered
### {Alternative}
- Summary
- Pros
- Cons
- Why not chosen

## References
- Related docs and sources
```

6. **Defaults.** Status is `Proposed` unless the user says it is decided. Include at least the alternative the user named plus any other serious option found in research.
7. **Finish.** Show the file path and a 2-3 line summary. Do not commit or push unless asked.
