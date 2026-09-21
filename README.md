# pa-spec

A specification for a minimal personal assistant that sits above all of your coding-agent sessions.

## The pitch

If you work with agent CLIs all day - Claude Code, pi, or any headless-capable agent CLI - you end up juggling sessions across models, losing context to window limits, and restating the same project facts every time. pa-spec describes the smallest system that fixes this: one CLI front door that owns the whole conversation, durable work state in SQLite, curated Markdown memory written by gated background passes, disposable worker sessions that can die and resume from the store, and a narrow trace plane that mines your own session history for agent failure patterns and your own usage patterns, treated as eval data. It is a spec, not a framework: roughly four weekends of work, one binary, no services, no daemons beyond scheduled passes.

## Who this is for

An individual engineer inside an organization that provides:

- an internal model gateway (any OpenAI-compatible endpoint) with **free tier models** for bulk work and **frontier tier models** for hard judgment calls;
- one or more headless-capable agent CLIs whose session transcripts you are allowed to read locally.

You want one agent above all your projects, not another heavy orchestration framework. You are willing to approve memory writes by hand at first in exchange for a corpus that does not rot.

This is explicitly not: a multi-user product, a knowledge graph, a desktop UI, or a universal cross-provider history importer. The trace plane is deliberately narrow - two verified transcript formats, four normalized outputs - and that narrowness is what keeps the build at four weekends.

## Design center

The hard part is not dispatch. It is the front-door agent's judgment: preserving one conversation, maintaining durable work state, handing off small packets of context, staying quiet when no interruption is useful, and learning without polluting memory. Every component in this spec exists to serve that judgment, and each one carries its rationale with it.

## Reading order

1. [spec/architecture.md](spec/architecture.md) - components, data flow, and the design principles everything else follows
2. [spec/storage.md](spec/storage.md) - the two-scope control plane (global + project SQLite)
3. [spec/session-wrapper.md](spec/session-wrapper.md) - headless dispatch, interactive wrap, exit harvest
4. [spec/trace-plane.md](spec/trace-plane.md) - trace schema, adapters, weekly pattern analysis
5. [spec/memory.md](spec/memory.md) - the write gate, note format, consolidation cadence
6. [spec/skills.md](spec/skills.md) - explicit, session-scoped instruction packs
7. [spec/routing.md](spec/routing.md) - routing.toml and judgment-level routing
8. [spec/proactivity.md](spec/proactivity.md) - watches, triggers, and the restraint rule
9. [spec/build-order.md](spec/build-order.md) - four weekends, each with acceptance criteria
10. [spec/open-questions.md](spec/open-questions.md) - environment facts to confirm before building

`config.example.toml` and `routing.example.toml` show the two configuration files with placeholder values. Your own gateway endpoints, model IDs, and CLI paths are environment-specific and do not belong in this repo.

## Status

Design reviewed against public systems (Letta/MemGPT, Zep/Graphiti, OpenClaw, Claude Code memory, mem0, Honcho, Supermemory, Obelisk as a studied reference) and against lessons from a working personal assistant. The 2026-09-18 mem0/Honcho evaluation earned the MemoryBackend interface, the candidate lifecycle, and the memory eval fixtures in [spec/memory.md](spec/memory.md). The 2026-09-18 turbopuffer evaluation added the `sqlite_hybrid` backend candidate, two-stage retrieval ranking, and the golden-query eval fixture to the same file. The 2026-09-21 Supermemory evaluation reached a not-adopted verdict: MIT/local and benchmark-worthy, but its MemoryBench phase checkpointing and quality/latency/context-token reporting are folded into the eval plan while its automatic graph writes stay behind the candidate/review gate - `supermemory_local` joins `mem0_oss` as an evaluation candidate only. Project skills (the SKILL.md convention with explicit, session-scoped selection) are specified in [spec/skills.md](spec/skills.md), written back from the weekend-3 build. A reference implementation is in progress against this spec. Feedback and adoption welcome; no license is attached yet, so treat it as all-rights-reserved until one is added.

