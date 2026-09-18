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
status: candidate|active|superseded|rejected|needs-review
stale_after: 2026-12-17
replaces: note:<id>|null
---
```

Machine-written notes also carry attribution fields, so retrieval can say who
recorded what about whom from which vantage point:

```
subject: user|project:<name>|<entity>   # who or what the note is about
observer: above-all|session:<id>        # whose perspective recorded it
project: <name>|null                    # project scope the note belongs to
session: <id>|null                      # originating session
evidence: [trace:<id>, outcome:<id>]    # backing records beyond the source anchors
```

A note about one observer's view is never presented as another's knowledge;
perspective leakage is an eval fixture, not a hope.

- `sources` is mandatory. A note that cannot say why it exists does not get admitted.
- `status` and `stale_after` are checked at read time. Retrieval filters out candidates, rejected, superseded, needs-review, and expired notes, and may abstain entirely when nothing active matches.
- The lifecycle is `candidate -> reviewed -> active -> superseded|rejected`. Sessions and background passes create candidates only; review (manual at first) promotes to active; replacement moves the prior note to superseded with its source chain intact; discarded candidates stay on disk as rejected for audit.
- `replaces` keeps the prior note and its source chain as `superseded` - replacement, never silent append-and-contradict.

FTS5 (SQLite full-text search) indexes only active, non-stale notes. At single-user scale, ranked FTS over a clean corpus beats embeddings over a dirty one.

## The write gate

Session exit creates *candidates*, never active memory. The admission pipeline:

1. Candidates are created from the session's result, corrections, and decisions - not from every message.
2. Deterministic checks reject candidates without source anchors, with empty claims, or duplicating an active note.
3. A free tier model classifies each candidate as discard, new, replace, or contradiction.
4. Promotion always runs a duplicate/conflict lookup against active notes first. Exact duplicates are discarded; overlaps become replace or contradiction. A promotion that skips this lookup is a bug.
5. Replace preserves the prior note and its source chain as superseded evidence - the old note stays on disk with its sources, and the new note links back through `replaces`.
6. Contradictions require human review. They are never merged silently.
7. Extraction is observable and fails loudly: an extraction error or an unexpectedly empty batch is recorded and surfaced as a warning, never silently dropped. Silent batch loss is a documented failure in comparable systems and is exactly what the gate exists to catch.
8. Human approval is the initial default for everything. Auto-admission may later be enabled per note type, only after measured precision over a review period and with easy rollback.

## Generated agent context

The generated `AGENT_CONTEXT.md` (consumed through your agent CLI's native project-context mechanism, e.g. CLAUDE.md) is built from a bounded set of approved project notes: stable guidance and links, not a dump of every note. Candidates and rejected notes never enter it.

The agent CLI's own auto-memory remains private scratch space. A consolidation pass may read it as a candidate source if the CLI exposes a stable location, but it is never auto-promoted to approved memory.

## Consolidation cadence

No large nightly rewrite. Three bounded passes:

- **Session exit**: create candidates, update outcome/session state. No active-memory mutation.
- **Daily sweep**: mark expired notes `needs-review` and drop them from the FTS index. No merging, no rewriting.
- **Weekly review**: process the candidate queue - approvals, duplicates, contradictions, stale notes - plus retrieval/write ratio review.

This separates fast operational durability from slow epistemic change, and keeps one bad summarizer run from rewriting the corpus overnight.

## The MemoryBackend interface

The knowledge plane sits behind a small interface so backends can be benchmarked against each other on real harvested traces before any swap:

- **admission**: create candidates (sessions and passes never write active memory)
- **retrieval**: ranked search over active, non-stale notes; abstention allowed
- **review**: approve, replace (supersede), discard
- **context**: the bounded approved set behind generated `AGENT_CONTEXT.md`

`sqlite_fts` - Markdown files plus an FTS5 index - is the reference backend. `mem0_oss` (Apache-2.0, self-hostable, configurable internal model/embedding/store endpoints) is the only pluggable candidate currently considered, and it is adopted only if replay evals on harvested traces show a clear retrieval win without worse pollution, provenance, latency, or operations. Its current OSS pipeline is single-pass ADD-only extraction with temporal preservation of changed facts; the older ADD/UPDATE/DELETE merge-loop descriptions are stale. Honcho is a study source, not a candidate: AGPL-3.0 is a hard stop, and its continuous latent inference is further from this gate than the current design. Its useful ideas - explicit observations vs inferred conclusions, observer-scoped representations, evidence-backed compact profiles, and a "why do you think this?" evidence query - are reflected in the attribution fields and eval fixtures, not in a dependency.

## Memory evals

Extraction and retrieval are evaluated separately, replaying an approved trace set against candidate backends before any backend change. Fixtures cover:

- **changed facts**: a later fact contradicts an earlier one; the corpus must present one current note and preserve the superseded evidence
- **duplication**: near-duplicate candidates must not produce parallel active notes
- **unsupported inference**: candidates with no source or evidence anchor must be rejected
- **cross-project leakage**: project notes must not surface in another project's retrieval or generated context
- **staleness**: expired notes must leave the index, and retrieval must abstain when nothing active matches
- **perspective leakage**: a note recorded from one observer's vantage must not be served as another's knowledge

A backend change ships only if it wins on these fixtures without worse pollution, provenance, latency, or operations.

## Pollution tripwires

Track, and act on:

- candidates created vs approved vs later retrieved
- notes replaced or contradicted within 30 days
- retrieved notes that were ignored or corrected
- generated `AGENT_CONTEXT.md` size
- prompt tokens spent on memory per completed outcome
- extraction failures and empty batches, which must always be visible (a silent zero is treated as a failure, not as "nothing to learn")

If write volume rises while approved-note use falls, stop auto-admission and tighten candidate generation. Consolidation never solves pollution by merging more aggressively.

