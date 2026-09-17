# Routing

Routing picks the cheapest model tier capable of each step. The unit of routing is the step's judgment level, not the task's label: a summary can contain a hard judgment, and a coding task can contain mechanical steps.

## routing.toml format

```toml
[steps]
filter = "free/example-small"
extract = "free/example-small"
summarize = "free/example-large"
consolidate = "free/example-large"
nonprod_execute = "free/example-large"

[[escalate]]
when = "high_stakes or prod_touching or ambiguity_high"
model = "frontier/example-primary"

[[escalate]]
when = "contradiction or independent_review_needed"
model = "frontier/example-secondary"
```

- `[steps]` maps step kinds to tier-qualified model IDs. The IDs are whatever your internal model gateway (any OpenAI-compatible endpoint) exposes; the values here are placeholders.
- `[[escalate]]` rules override the step table when the condition holds. Conditions are evaluated from step metadata: stakes, whether the output touches production, classifier confidence, contradiction flags.
- A classifier (free tier) recommends the route; hard safety and production rules always override it.

## Rules

- Escalation happens per step, based on confidence, stakes, and verification needs - not once per task.
- Free tier is the default for filtering, extraction, summarization, consolidation, and non-production loops. Frontier tier is reserved for ambiguity, difficult synthesis, and production-sensitive judgment.
- `routing.toml` is hand-editable. The weekly trace-analysis pass may *suggest* changes (with evidence) but never edits the file itself.
- Keep the step vocabulary small. Five to eight step kinds cover a personal assistant; new kinds earn their place through trace evidence, not anticipation.

See `routing.example.toml` at the repo root for a copyable file.
