# Why bigger models will not save you — three facts about LLM runtime

*Appendix to [README.md](../../README.md). The mechanical answer to "why is the ceiling 1M and not 1T" — and why a bigger window does not change the shape of the problem.*

You do not need deep ML to place knowledge well in Claude Code. You need three facts, in just-enough form. Every placement decision in the README flows from them.

## Fact 1 — The context window is finite, and shared

Opus 4.7 tops out at 1M tokens. That sounds enormous (huge to anyone who has read [self-attention](./self-attention.md), but a rounding error next to RAM or disk capacity — if context ever reached TB territory like memory does, we'd all be farming for a living by now); it is not, because the budget is **shared**, not dedicated to your question. Everything below competes for the same 1M on every single turn:

- Claude Code's own system prompt
- Every tool schema — built-in, MCP, agent definitions
- Every `CLAUDE.md`, every always-loaded rule, every skill description, `MEMORY.md`
- The entire prior conversation, every tool result, every file Claude has read

A single read of a 2000-line source file can eat 30–50k tokens. Ten such reads in a long debugging session and you have burned 5% of the window before doing any actual work. "1M tokens" is the ceiling, not the workspace — the usable workspace shrinks every minute of a session.

## Fact 2 — Every request re-sends the entire conversation

A finite window would be manageable if the model kept state between turns. It does not. LLM inference is **stateless** — the client re-sends the full prior conversation, all loaded `CLAUDE.md`, all rules, all skill descriptions, all tool definitions, **every single turn**, from turn 1.

A practical consequence: a 500-line addition to `CLAUDE.md` is not a one-time cost. It is a per-turn tax, multiplied by every turn of every future session. Prompt caching softens the blow (0.1× for matching prefixes within a 5-minute TTL), but caching only helps if you do not invalidate the prefix — and editing `CLAUDE.md` mid-session invalidates everything downstream.

## Fact 3 — Bigger context does not mean better focus

Paying per-turn would also be manageable if every token you paid for worked equally hard. It does not. Self-attention — the mechanism the model uses to decide which tokens matter — gives every token a weight, and **those weights must sum to a fixed total**. Every additional irrelevant token reduces the weight available for the relevant ones. More context makes the signal thinner, not sharper. Combined with the empirically measured "lost in the middle" effect (info buried between long prefixes and long suffixes gets weighted less), a prompt twice as long can be *less* effective than a prompt half the size, if the extra content is noise.

> **Deep dive:** [`self-attention.md`](./self-attention.md) animates why this is mechanical, not a bug a bigger window fixes. [`why-context-matters.md`](./why-context-matters.md) expands all three facts with examples of what each one costs you in practice.

## The trajectory has a floor

These three facts do not get fixed by a bigger window. They compound with it. Models keep getting smarter; context windows keep growing — 8k → 200k → 1M → whatever comes next. Engineering wins like prompt caching, path-scoped rules, compaction, and subagent isolation all push the ceiling up. But they do not change the *shape* of the ceiling: **the problem scales with your ambition, not with the model.** The day a 10M-token window arrives, you will find yourself wanting to pack 20M tokens of project context into it.

## Reasoning is not the same as durable knowledge

One more fact, this one observational. I once applied a Terraform config that failed. With no hand-holding, Claude inferred the URL might be wrong, went looking for the provider's source, could not find it locally (the provider was named against convention), web-searched the closest match, found the GitHub repo, read the source, and discovered a trailing `/` that should not be there. Mind-blowing. And of course I did not dare close that session — surely that expertise lived *somewhere*. Sure enough, N turns of unrelated work later, a similar issue cropped up and Claude had to re-derive half of it from scratch.

A model smart enough to *derive* the right answer once will still re-derive it, imperfectly, every time its context drifts or resets. In a long session, attention rot eventually buries the answer. In a fresh session, it is never there to begin with.

> **Deep dive:** [`delegation-ceiling.md`](./delegation-ceiling.md) for the full war story and why better models alone do not close the gap.
