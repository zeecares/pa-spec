# Proactivity: schedules, triggers, restraint

A personal assistant that never speaks first is a tool; one that interrupts freely is a pest. The design target is between: watches emit internal events, and a value gate decides whether any of them becomes an interruption.

## Mechanics

- **Watches** live in the control plane (`watches` table): a kind, a spec, a next-fire time, and a per-watch interruption policy. Kinds include schedules (cron-like), event triggers (a watched source changed), and deadlines (one-shot timeouts on real commitments).
- **Events** are the internal record. A firing watch writes an event; it does not message the user.
- **The value gate** reviews pending events at natural interaction points (and on a bounded cadence) and applies the restraint rule.

## The restraint rule

A proactive message must name a concrete decision, a risk, or a step it saves the user. If it cannot, it stays in the internal queue until the user's next natural interaction, where it surfaces as context, not as an interruption.

Consequences:

- Acknowledgment is not proactivity. Accepting a task gets an instant receipt (accepted/running/blocked/done) - that is table stakes, not an interruption.
- Batch the trivial. Several low-value events become one line in the next reply, not five pings.
- Silence is a success state. A week with no proactive message and no missed risk means the gate is working.
- The weekly trace analysis follows the same rule: quiet unless it has an actionable finding.

## Why the gate is separate from the watches

Triggers are easy and their number only grows - every integration and schedule adds more. If interruption policy lives inside each watch, every new watch is a new way to annoy the user. One gate, one policy, every watch measured against it.
