# Vision

**Scope:** pa-spec, the specification for a personal assistant that sits
above your coding-agent sessions, and the rules for how the spec changes. The
reference implementation follows this document and the spec. It does not
lead them.

This is not an instruction file for agents. Agents read the spec sections.
This file says what the project is trying to become, what it will not
become, and which changes can land without the owner.

## Where it is going

One engineer, many agent sessions, one front door that keeps what matters
and lets the rest go. Success looks like this:

- A reader can build the system from the spec in about four weekends
  without guessing.
- Every active memory note can answer two questions: where did this come
  from, and is it still true.
- The assistant stays quiet unless it can name the decision, risk, or saved
  step behind an interruption.
- Swapping the memory backend is an experiment behind one interface, never
  a rewrite.
- The spec gets smaller or stays the same size as it gets better.

## Non-goals

These are settled. Reopening one needs a written reason and the owner's
sign-off.

- Not a multi-user product or a hosted service.
- Not a knowledge graph or vector store as the source of truth.
- Not a desktop UI.
- Not a universal history importer. Only verified transcript formats.
- Not an autonomous planner that recurses into its own sub-plans.
- Model output never becomes active memory without review.
- No vendor names, internal endpoints, model IDs, or credentials in the
  repo.
- No AGPL dependencies.

## Merge by default

A change merges without waiting for the owner when all of these hold:

- CI is green on the full matrix.
- An adversarial review pass ran, and its findings are written in the pull
  request.
- Every deterministic gate passes: tests, lint, fixture checksums, schema
  validation, provenance labels.
- The change is one of: tests or fixtures, typo and link fixes, docs that
  describe existing behavior, placeholder config examples, or a status line
  recording a decision the owner already signed off.

The owner hears about it afterwards, with what the review caught.

## Needs sign-off

These come to the owner before merging:

- Any change to a design principle in `spec/architecture.md` or to a
  non-goal above.
- Storage schema, note format, or migration changes.
- Write-gate, provenance, or memory-lifecycle rules.
- Routing or proactivity policy.
- A new dependency, backend, or license term.
- Anything that spends money, sends messages, or touches external accounts.
- Anything the review marks as novel or low-confidence.

When in doubt, it needs sign-off.

## Keeping the line honest

- If a default merge surprises the owner, the merge-by-default list gets
  shorter.
- Every issue a review catches becomes a permanent regression test, so later
  reviews only see new kinds of risk.
