# Skills: explicit, session-scoped instruction packs

A skill is a named pack of instructions the front door can hand to a session on request - how to run a deploy, how this team writes migrations, a checklist for a review. Skills extend what a session knows how to do without growing the standing prompt of every session.

The mechanism follows the emerging SKILL.md convention (one directory per skill, a Markdown file with a small YAML header, relative references to supporting files) so skills written for agent CLIs can be reused here unchanged.

## Format

```
skills/
  deploy/
    SKILL.md
    rollback.md        # optional supporting file
```

```markdown
---
name: deploy
description: Deploy or roll back the staging environment; use when asked to ship to staging.
---

Steps...
```

- One directory per skill; the directory name must match `name`.
- The frontmatter carries `name` and `description` only. The description doubles as the when-to-use trigger a person (or a model suggesting a skill) reads to judge relevance.
- The body is Markdown. A skill may reference sibling files with relative paths; the wrapper resolves them relative to the SKILL.md location when it packs the handoff.

## Discovery and shadowing

Skills live in both scopes: `~/.assistant/skills/` global and `<project>/.assistant/skills/` project. A project skill shadows a global skill with the same name, so a project can specialize a shared procedure without forking the global one. Listing skills shows the merged set with the winning path, so shadowing is visible rather than surprising.

## Explicit selection only

A session receives a skill only when it is named: `--skill deploy` on `assistant do` or `assistant work`. There is no keyword matching, embedding match, or automatic injection. A model may suggest a skill for a task, but a person - or an explicit rule a person wrote - applies it. Naming an unknown skill fails loudly instead of being silently dropped.

Rationale: skills are instructions, not authority. Automatic selection would grow the effective prompt by whatever a matcher fired on, unaudited - the same uncontrolled-retention failure the memory write gate exists to prevent, on the instruction side. Explicit selection keeps every injected instruction inspectable in the session record.

## Session scoping

A selected skill lives for the session that asked for it. Headless sessions get the skill text inside the handoff envelope; interactive sessions get a `SELECTED_SKILLS.md` written into the session directory and exposed through the session environment. Nothing writes a selection to project or global state, so a later session started without `--skill` provably sees no residue of an earlier one.
