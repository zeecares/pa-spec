# Open questions and pre-build checks

These are environment facts, not design decisions. Confirm them before or during Weekend 1; several can block Weekend 3 entirely.

## Transcripts

- What exact transcript formats and locations do your agent CLI builds write? Do stock builds differ from public documentation?
- Do they expose stable session IDs, compaction markers, branch/supersession markers, subagent parent IDs, tool results, and token counts?
- Can the wrapper reliably identify the session artifact created by the process it launched, including under concurrent sessions in the same project?
- Do your organization's data-handling rules permit copying or indexing transcript files, even locally? (This decides whether the trace plane exists at all.)

## Headless operation

- Which headless and system-prompt flags are available on your agent CLI builds?
- What is the verified headless invocation (command, flags, output format) the wrapper should use?

## Model gateway

- Can the CLI reach your internal model gateway from the machines where the assistant will run?
- What are the auth mechanism, rate limits, context limits, and concurrency limits for the free tier and frontier tier models you plan to route to?

## Project state

- How should project `.assistant/notes` split between commit-safe team knowledge and private local knowledge? Per project, who decides?
- Does your agent CLI's auto-memory expose a stable readable location, and should it be sampled only as a candidate source?

## Review budget

- What is the initial human-review budget for memory candidates: per session, a daily queue, or a weekly batch? Pick the one you will actually sustain; the write gate only works if review happens.
