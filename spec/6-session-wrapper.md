# Session wrapper

The wrapper is the only component that launches agent-CLI processes. It exists so every session - headless or interactive - leaves durable state behind when it dies.

## Headless dispatch

1. `assistant do "<intent>"` creates an outcome and returns acceptance immediately (acknowledgment rhythm: the user gets a task ID and status, not silence).
2. The dispatcher builds a small handoff: raw intent, constraints, source anchors, the active relevant notes, and the expected return shape.
3. Routing selects the cheapest model tier capable of each step (see [routing.md](routing.md)).
4. The wrapper launches the agent CLI's verified headless command (for example `your-agent-cli -p "<prompt>" --output-format json`; exact flags are environment-specific - see [open-questions.md](open-questions.md)).
5. On exit it records result status and evidence, summarizes the session through a free tier model, and proposes memory candidates.
6. The trace adapter imports the verified normalized subset, if the provider's transcript format is supported (see [trace-plane.md](trace-plane.md)).

## Interactive wrap

The user must be able to drop into a live agent-CLI session, not just dispatch background workers.

1. `assistant work` resolves the project and regenerates the bounded `AGENT_CONTEXT.md` from approved project notes.
2. The wrapper launches the agent CLI normally; the user drives it directly.
3. On exit the wrapper finds the exact session artifact the launched process produced, records metadata and a summary, proposes memory candidates, and imports traces through the matching adapter.
4. A failed import does not lose the session outcome. It creates a visible internal warning and leaves the source transcript untouched.

## Handoff envelope

```json
{
  "outcome_id": "...",
  "intent": "raw user intent",
  "constraints": [],
  "source_anchors": [],
  "relevant_notes": [],
  "return": ["outcome", "evidence", "changes", "blockers"]
}
```

Rules:

- Raw intent, not a paraphrase chain. Summaries compound distortion; the worker reads what the user actually asked.
- Context is capped. `relevant_notes` is a bounded set selected by the front door, not a dump of the knowledge plane. The front door owns reconstructing the user-facing answer from the return packet.
- The return shape is fixed so completion always means outcome plus evidence plus changes plus blockers.

## Session identity

The wrapper must reliably identify the transcript artifact created by the process it launched, including under concurrent sessions of the same provider in the same project. Prefer launching with an explicit session/output path when the CLI supports it; fall back to matching on creation time and parent PID, and treat ambiguous attribution as an import warning, never a guess. Parent-child relationships (an agent CLI spawning subagents) are recorded via `parent_session_id` when the transcript format exposes them.

## Exit harvest ordering

On any session exit, in order: update outcome/session state -> write the exit summary -> create memory candidates -> import traces. The first two steps must succeed even if everything after them fails. Session exit never writes active memory directly; it creates candidates (see [memory.md](memory.md)).
