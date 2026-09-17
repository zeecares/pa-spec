# Trace plane

Your own session history is the cheapest eval data you have. The trace plane imports a normalized subset of every agent-CLI session into SQLite so a weekly analysis pass can mine it for agent failure patterns and your own usage patterns.

Scope discipline: this is deliberately not a universal session indexer. Two verified transcript formats, four normalized outputs. Obelisk (https://github.com/tommy0103/obelisk) demonstrates what a full cross-provider index looks like and is worth studying; it is AGPL-3.0, so it is a reference here, not a dependency, and none of its code is used.

## What Obelisk taught us (as design knowledge)

- Transcript parsing is an adapter boundary, not application logic.
- Compacted or superseded branches must not be counted as ordinary live history.
- Parent-child session identity matters when a coding agent creates subagents.
- Incremental import needs stable source identifiers and idempotency.
- Tool failures and token use are useful eval signals only when tied to a session and an outcome.
- Human-approved memory proposals are safer than automatic retention.

What not to copy: no cross-provider universal index, no desktop UI, no workflow reconstruction, no full subagent graph in the first build, and no promise to support every historical transcript variation.

## Schema

Full transcripts stay in their provider-owned location. SQLite stores only the normalized subset needed for analysis.

```sql
CREATE TABLE trace_messages (
  source_id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  seq INTEGER NOT NULL,
  role TEXT NOT NULL,
  ts TEXT,
  text TEXT,
  active_branch INTEGER DEFAULT 1
);

CREATE TABLE trace_tool_calls (
  source_id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  seq INTEGER NOT NULL,
  tool_name TEXT NOT NULL,
  ts TEXT,
  status TEXT,                 -- ok|failed|unknown
  error_class TEXT,
  duration_ms INTEGER
);

CREATE TABLE trace_usage (
  session_id TEXT PRIMARY KEY,
  tokens_in INTEGER,
  tokens_out INTEGER,
  cached_tokens INTEGER,
  reported_cost_usd REAL,
  source TEXT NOT NULL
);

CREATE TABLE trace_imports (
  source_path TEXT PRIMARY KEY,
  provider TEXT NOT NULL,
  format_version TEXT,
  fingerprint TEXT NOT NULL,
  imported_at TEXT NOT NULL,
  status TEXT NOT NULL,
  warning TEXT
);
```

Rules:

- Import is idempotent on stable source ID plus fingerprint. Re-importing the same transcript is a no-op; a changed fingerprint is a warning, not a silent overwrite.
- Unsupported records are logged, never guessed into the schema.
- Compaction, branches, and subagent parentage are adapter test fixtures before they become analytics inputs. A compacted branch marked inactive must not double-count messages or tokens.
- One adapter per provider, enabled only after that provider's stock transcript format is verified against real fixtures. If a provider's format is incompatible or unstable, the wrapper still records session metadata and result summaries while trace import stays disabled for that provider.

## Weekly pattern analysis

Run when enough new completed outcomes exist - not merely because seven days passed. Use SQL for counts and joins, then a free tier model for grouping and explanation.

Useful signals:

- repeated tool failures by provider, tool, and project
- repeated context the user had to restate
- corrections issued after an answer or code change
- escalation from free tier to frontier tier, and whether it helped
- token use per completed outcome (not per session - a long session may be justified, a cheap one may produce rework)
- abandoned, blocked, or reopened outcomes
- recurring manual steps that may deserve automation

Outputs are three small queues, each item carrying sessions/outcomes as evidence, an estimated value, and a proposed experiment:

1. **System candidates**: parser gaps, routing changes, tool reliability issues, missing context.
2. **User candidates**: repeated manual patterns or habits worth reviewing.
3. **Eval candidates**: representative successes, failures, corrections, and regressions worth pinning as test cases.

Nothing modifies routing, memory admission, or anything else automatically. The analysis proposes; a human applies. It should be quiet when it has no actionable finding - a weekly dashboard is optional, and interruption requires a decision, a risk, or a clear saved effort (see [proactivity.md](proactivity.md)).
