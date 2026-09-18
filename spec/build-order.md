# Build order: four weekends

Each weekend ends with something that works and an acceptance criterion that proves it. Do not start the next weekend's scope early; the order is chosen so each layer earns the next.

## Weekend 1 - front door and work state

- CLI skeleton and project resolution (global scope + project overlay)
- Control-plane schema from [storage.md](storage.md)
- `assistant do`: outcome creation, ownership, event log, immediate accepted/running/blocked/done status
- One verified headless dispatch path through the session wrapper

**Acceptance:** delegate one background outcome, kill everything, and recover its status from the store without replaying any transcript.

## Weekend 2 - project memory with the write gate

- Note format and OKF-lite headers from [memory.md](memory.md)
- FTS5 index with read-time staleness filtering
- Candidate queue and the manual approve/replace/discard flow
- Generated `AGENT_CONTEXT.md` from a bounded approved set
- Exit summaries through a free tier model

**Acceptance:** two sessions share approved project knowledge without transcript replay and without a second, parallel memory system inside the agent CLI.

## Weekend 3 - minimal trace adapters and first analysis

- Inspect and fixture the actual transcript formats of your agent CLI builds (see [open-questions.md](open-questions.md))
- Implement the easier verified adapter first; the second only if its format proves stable
- Normalize messages, tool calls, failures, and token usage only
- Idempotency and compaction/branch fixtures
- First SQL-driven pattern report tied to outcomes
- Minimal skill loader from [skills.md](skills.md): SKILL.md discovery across both scopes, explicit `--skill` selection only

**Acceptance:** one provider's adapter fully works and is tested. Two providers is a stretch goal; a second adapter is conditional on format stability, not promised.

## Weekend 4 - restrained proactivity and consolidation

- Daily expiry sweep and bounded weekly consolidation
- Watches and events with the interruption value gate
- `routing.toml` live, with trace-informed (human-applied) suggestions
- Pollution, retrieval-recall, prompt-size, and free/frontier cost report
- Recovery tests: killed session, failed import, stale note, contradictory note

**Acceptance:** the assistant holds work across sessions, learns only through review, and stays quiet when no action is useful.

## Sizing note

Four weekends holds only because trace capture is deliberately narrow. Universal history import, desktop browsing, full subagent trees, or additional providers would move trace work into a separate project. Guard that boundary; it is the difference between a spec that gets built and one that gets admired.

