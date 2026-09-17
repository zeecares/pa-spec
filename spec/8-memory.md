# Memory: the knowledge plane and its write gate

Knowledge memory holds facts, preferences, decisions, and procedures that may help later. It is separate from work state (outcomes, owners, blockers - see [storage.md](storage.md)). A session summary may point to an outcome; it never becomes the outcome record.

The known failure mode of agent memory is uncontrolled retention: if every session emits durable facts, contradictions and prompt cost grow faster than usefulness, and no retrieval technique fixes a polluted corpus. So this spec spends its complexity on admission control, not retrieval.

## Note format

Approved notes are Markdown files with a small provenance header (an OKF-lite header - six fields borrowed from Google's Open Knowledge Format, without adopting the graph, catalog server, or attestation machinery):

```
---
type: decision|fact|preference|procedure|summary
sources: [session:<id>, outcome:<id>, file:<path>]
generated: 2026-09-17T08:00:00+01:00
verified: human|source|none
status: active|draft|superseded|needs-review
stale_after: 2026-12-17
replaces: note:<id>|null
---
```

- `sources` is mandatory. A note that cannot say why it exists does not get admitted.
- `status` and `stale_after` are checked at read time. Retrieval filters out drafts, superseded, needs-review, and expired notes, and may abstain entirely when nothing active matches.
- `replaces` keeps the prior note and its source chain as `superseded` - replacement, never silent append-and-contradict.

FTS5 (SQLite full-text search) indexes only active, non-stale notes. At single-user scale, ranked FTS over a clean corpus beats embeddings over a dirty one.

## The write gate

Session exit creates *candidates*, never active memory. The admission pipeline:

1. Candidates are created from the session's result, corrections, and decisions - not from every message.
2. Deterministic checks reject candidates without source anchors, with empty claims, or duplicating an active note.
3. A free tier model classifies each candidate as discard, new, replace, or contradiction.
4. Replace preserves the prior note and its source chain as superseded.
5. Contradictions require human review. They are never merged silently.
6. Human approval is the initial default for everything. Auto-admission may later be enabled per note type, only after measured precision over a review period and with easy rollback.

## Generated agent context

The generated `AGENT_CONTEXT.md` (consumed through your agent CLI's native project-context mechanism, e.g. CLAUDE.md) is built from a bounded set of approved project notes: stable guidance and links, not a dump of every note. Drafts never enter it.

The agent CLI's own auto-memory remains private scratch space. A consolidation pass may read it as a candidate source if the CLI exposes a stable location, but it is never auto-promoted to approved memory.

## Consolidation cadence

No large nightly rewrite. Three bounded passes:

- **Session exit**: create candidates, update outcome/session state. No active-memory mutation.
- **Daily sweep**: mark expired notes `needs-review` and drop them from the FTS index. No merging, no rewriting.
- **Weekly review**: process the candidate queue - approvals, duplicates, contradictions, stale notes - plus retrieval/write ratio review.

This separates fast operational durability from slow epistemic change, and keeps one bad summarizer run from rewriting the corpus overnight.

## Pollution tripwires

Track, and act on:

- candidates created vs approved vs later retrieved
- notes replaced or contradicted within 30 days
- retrieved notes that were ignored or corrected
- generated `AGENT_CONTEXT.md` size
- prompt tokens spent on memory per completed outcome

If write volume rises while approved-note use falls, stop auto-admission and tighten candidate generation. Consolidation never solves pollution by merging more aggressively.
