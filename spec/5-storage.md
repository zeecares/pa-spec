# Two-scope storage

State lives in two scopes that the CLI merges at runtime:

- **Global** (`~/.assistant/`): cross-project outcomes, preferences, people, global watches, the trace index. Never committed anywhere.
- **Project** (`<project>/.assistant/`): project outcomes, decisions, approved project notes, the generated agent-context file. May travel with the repo.

Inside a project directory the CLI resolves the repo root, opens the global database, then overlays the project database. Outside a project it works from global scope alone. Queries that need both scopes join explicitly; nothing silently bleeds across.

## Commit-safe vs private

Project state splits by audience:

- **Commit-safe**: approved notes that are team knowledge, the generated context file. These may be committed if the team wants shared agent context.
- **Private**: candidates, session metadata, trace rows, anything machine-local or personal. Keep these out of version control (`.assistant/` entries in `.gitignore` except an explicit allowlist), or keep the whole project scope local-only if your environment's data-handling rules require it.

Decide this split per project before the first build; it is one of the open questions because the answer is policy, not code.

## Control-plane schema (per scope)

```sql
CREATE TABLE outcomes (
  id TEXT PRIMARY KEY,
  project TEXT,
  title TEXT NOT NULL,
  status TEXT NOT NULL,       -- pending|running|blocked|done|cancelled
  owner TEXT NOT NULL,        -- frontdoor|session:<id>|user
  source_anchor TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE sessions (
  id TEXT PRIMARY KEY,
  provider TEXT NOT NULL,     -- which agent CLI produced this session
  project TEXT,
  mode TEXT NOT NULL,         -- interactive|headless
  model TEXT,
  outcome_id TEXT,
  parent_session_id TEXT,
  started_at TEXT,
  ended_at TEXT,
  source_path TEXT,
  import_version TEXT,
  summary TEXT,
  tokens_in INTEGER,
  tokens_out INTEGER,
  cost_usd REAL
);

CREATE TABLE decisions (
  id TEXT PRIMARY KEY,
  ts TEXT NOT NULL,
  project TEXT,
  outcome_id TEXT,
  decision TEXT NOT NULL,
  rationale TEXT,
  source_anchor TEXT
);

CREATE TABLE events (
  id INTEGER PRIMARY KEY,
  ts TEXT NOT NULL,
  kind TEXT NOT NULL,
  outcome_id TEXT,
  payload_json TEXT
);

CREATE TABLE watches (
  id TEXT PRIMARY KEY,
  kind TEXT NOT NULL,
  spec_json TEXT NOT NULL,
  next_fire TEXT,
  active INTEGER NOT NULL,
  interruption_policy TEXT NOT NULL
);
```

Notes:

- `outcomes` is the unit of delegated work. Owner and status are mandatory; completion means an outcome plus evidence, not a summary. `source_anchor` points at the originating message, file, or session so any state can be traced back to why it exists.
- `sessions` records every launched or wrapped agent-CLI session, including ones whose trace import later fails. A failed import never loses the session row or the outcome.
- `events` is the append-only log behind the acknowledgment rhythm: accepted, running, blocked, done are event kinds the CLI can print instantly.
- `watches` backs proactivity (see [proactivity.md](proactivity.md)); `interruption_policy` is stored per watch, not global.
- IDs are stable, human-inspectable strings. Imports and replays are idempotent on them.

The knowledge plane (notes, FTS5, the candidates queue) is specified in [memory.md](memory.md).
