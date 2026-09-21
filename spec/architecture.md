# Architecture overview

One binary, one front door, two storage scopes, disposable sessions. The front door is the only component that talks to the user; everything else is a worker, a store, or a scheduled pass.

## Components

```
                 user
                   |
              CLI front door        <- owns the conversation, status, judgment
                   |
      +------------+-------------+
      |                          |
  control plane             knowledge plane
  (SQLite: outcomes,        MemoryBackend interface:
   sessions, decisions,      Markdown notes + FTS5,
   events, watches)          gated writes, read-time
                             staleness filtering)
      |                          |
      +------------+-------------+
                   |
            session wrapper     <- launches agent CLI sessions, harvests on exit
                   |
        +----------+----------+
        |                     |
   headless workers      interactive sessions
   (agent CLI -p)        (user drops in live)
        |                     |
        +----------+----------+
                   |
              trace plane     <- adapters import a normalized subset on session exit
                   |
        weekly analysis pass  <- evidence-backed candidates, never auto-applied
```

Background passes (session-exit, daily sweep, weekly consolidation, weekly trace analysis) run on a schedule and propose; they never interrupt on their own and never edit policy automatically.

## Design principles

These come from watching a working above-all personal assistant operate, and every spec section points back to one of them.

**Conversation ownership is the real orchestration job.** The front door owns continuity and judgment: when to answer, delegate, ask, acknowledge, or stay quiet. Knowing what not to do or surface matters as much as dispatch. This quality comes from iteration and evals, not from a richer worker graph. Keep orchestration policy explicit and inspectable; start with a few rules; retain decision traces; do not build autonomous planner recursion.

**Durable work state is separate from knowledge memory.** Work state answers: what outcome is open, who owns it, what blocks it, where is the evidence. Knowledge memory answers: what fact, preference, decision, or procedure may help later. Putting both in one note pile makes tasks disappear into prose and old status get retrieved as truth. Every delegated unit gets an owner and a status; completion requires an outcome plus evidence, not merely a generated summary.

**Handoffs stay small.** Workers need raw intent, constraints, relevant source anchors, and only the minimum context required - never whole transcripts. Their return packet is outcome, evidence, changes made, unresolved blockers. The front door reconstructs the user-facing answer; this is what keeps the front door light.

**Acknowledgment rhythm makes async work feel held.** A fast receipt followed by a substantive answer when ready is cheap and changes the experience. Without it, delegation feels like disappearance. The CLI records and prints an accepted task ID immediately; status distinguishes accepted, running, blocked, and complete.

**Sessions are disposable.** Compaction and death are routine, not exceptional. Every session must be able to end and resume from durable state; a transcript is evidence, not the only copy of the plan. Persist tasks, decisions, source anchors, and result summaries at safe boundaries; never require replaying a transcript to recover work.

**Proactivity is triggers plus restraint.** Schedules and event triggers are the easy half. The hard half is interruption policy: a proactive message must name a concrete decision, risk, or saved step, or it stays in the internal queue until the next natural interaction.

**The knowledge plane hides behind a MemoryBackend interface.** SQLite plus FTS5 is the reference backend. `sqlite_hybrid`, `mem0_oss`, and `supermemory_local` are evaluation candidates; none may replace the reference backend until it wins replay evals on harvested traces without weakening the write gate, provenance, isolation, latency, cost or operations. Honcho remains a study source because AGPL is a hard stop. Backends are swappable; the write gate is not.

**Memory writes happen off the critical path.** The session that does the work does not also decide what becomes permanent truth. Background passes propose memory after the outcome is known. Machine-written notes start as candidates, replace rather than append, keep their sources, and are filtered for staleness at read time. Uncontrolled retention is the documented killer of agent memory - not bad retrieval - so admission control outranks embeddings, graphs, and every other retrieval upgrade.

**Route by judgment level, not task label.** A summary can contain a hard judgment; a coding task can contain mechanical steps. The cheapest capable model handles each step, with escalation on confidence, stakes, and verification needs.

## Directory layout

```
~/.assistant/
  assistant.db              # global control plane and trace index
  notes/                    # approved cross-project knowledge
  candidates/               # machine-written memory candidates awaiting review
  routing.toml              # editable routing and escalation rules
  config.toml               # gateway, tiers, CLI paths, schedules
  adapters/
    <provider>_trace.py     # one per verified transcript format
  skills/                   # global instruction packs (see skills.md)
  logs/

<project>/.assistant/
  assistant.db              # project outcomes, decisions, note index
  notes/                    # approved project knowledge
  candidates/               # draft project notes
  skills/                   # project instruction packs; shadow global ones by name
  AGENT_CONTEXT.md          # generated native context surface for the agent CLI
```

One binary resolves the current repository, loads global state, then overlays project state. Project notes may travel with the repo only if their contents are appropriate to commit; private or machine-local notes stay global or in an ignored local file (see [storage.md](storage.md)).

`AGENT_CONTEXT.md` is the generated, bounded import surface consumed by your agent CLI's native project-context mechanism (CLAUDE.md in Claude Code, or the equivalent elsewhere). Generating a bounded file through the CLI's own mechanism - instead of running a parallel invisible memory store - avoids two contradictory memory systems inside one session, which is worse than one mediocre one.

## Provider neutrality

The spec names no vendor. Wherever a concrete choice is needed:

- **agent CLI**: any headless-capable coding/agent CLI (Claude Code, pi, or similar)
- **model gateway**: any OpenAI-compatible endpoint
- **model tiers**: "free tier" for bulk/filter/summarize work, "frontier tier" for ambiguity, difficult synthesis, and production-touching judgment

`config.example.toml` and `routing.example.toml` carry placeholder values; map them to whatever your gateway exposes.

