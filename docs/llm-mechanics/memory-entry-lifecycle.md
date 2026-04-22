# The lifecycle of each `/memory` entry

*What gets loaded when, what survives `/compact`, and what Anthropic's prompt cache actually stores.*

`/memory` gives you a snapshot of everything currently loaded from disk into your session's context: every `CLAUDE.md`, every loaded rule, every auto-memory file. What it does not tell you is that **the items in that snapshot do not all live the same way**. Knowing how each one loads, survives compaction, and enters the prompt cache is what separates *"stuff Claude reads once"* from *"stuff Claude keeps paying for on every turn."*

This doc is the reference table for that. It is intentionally short and load-bearing — the main README links here whenever it needs the details.

---

## The table

| What you see in `/memory`                                           | When it enters context                                  | What happens after `/compact`                                                                                | In the prompt cache?                                                  |
| :------------------------------------------------------------------ | :------------------------------------------------------ | :----------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------- |
| Project / user `CLAUDE.md`                                          | Once at session start                                   | Re-injected from disk                                                                                        | Yes — part of the 5-minute (optionally 1-hour) prefix cache           |
| Always-loaded rules (`.claude/rules/*.md` without `paths:`)         | Once at session start                                   | Re-injected from disk                                                                                        | Same prefix as `CLAUDE.md`                                            |
| Auto memory (`MEMORY.md`)                                           | Once at session start (first 200 lines / 25KB)          | Re-injected from disk                                                                                        | Same prefix                                                           |
| Skill **descriptions** (metadata only)                              | Once at session start                                   | **Lost.** The skill listing is the one block not re-injected — only the skills you actually invoked survive  | Same prefix (while still in context)                                  |
| Path-scoped rules (`paths: src/api/**`)                             | When Claude reads a matching file, mid-session          | Lost until a matching file is read again                                                                     | Enters cache only after first injection                               |
| Nested `CLAUDE.md` in subdirectories                                | When Claude reads a file in that subdirectory           | Lost until a file in that subdirectory is read again                                                         | Same as path-scoped rules                                             |
| Memory topic files (everything in `memory/` other than `MEMORY.md`) | Only when Claude explicitly reads one                   | Only survives if already read                                                                                | Enters cache only after first injection                               |
| Skill **bodies**                                                    | On invocation (by you via `/name` or by Claude)         | Re-injected, capped at 5,000 tokens per skill and 25,000 tokens total; oldest dropped first                  | Enters cache only after first injection                               |
| Hooks                                                               | N/A — hooks run in the harness, not in context          | N/A                                                                                                          | Never in context, never cached                                        |

---

## Two non-obvious consequences

### 1. "Loaded at session start" is not a single category

`CLAUDE.md`, always-loaded rules, auto-memory, and skill descriptions all appear before your first prompt. It is tempting to think of them as one group. They are not. Skill descriptions are the one thing `/compact` silently drops. The consequence has a subtle asymmetry that matters the moment you start relying on skills:

- **Explicit `/command` still works after `/compact`.** Typing `/skill-name` goes through the Claude Code harness (the client), which loads the skill body directly from disk and injects it into the conversation. Claude's in-context knowledge of the skill is not required to route a `/command` — the harness owns the menu.
- **Implicit auto-trigger is broken**, for any skill Claude has not already invoked in this session. Claude no longer sees the description list, so it does not know those skills exist. Any `disable-model-invocation: true` skill is unaffected (its description was never in context in the first place); anything else that relied on Claude auto-routing silently stops firing.
- **Net effect:** your "magic" auto-triggering framework feels noticeably dumber after a long session — not because the model got worse, but because half its routing map was summarized away. This is one of the most common *"Claude used to do X automatically, now it doesn't"* complaints, and it always has the same cause.

### 2. Path-scoped rules and nested `CLAUDE.md` are the most fragile entries

Both of those live inside the message history rather than the prefix. Compaction erases them, and they only come back when Claude happens to read a matching file again. If a rule must persist through compaction, remove the `paths:` scope or hoist the content into the project-root `CLAUDE.md`. Otherwise, accept that the rule's presence is conditional on Claude's recent file reads.

---

## Sources

Worth a direct read — especially the first one, which animates a lot of this visually:

- [Claude Code · Explore the context window](https://code.claude.com/docs/en/context-window) — interactive timeline of a simulated session, including the *"What survives compaction"* table.
- [Claude Code · Memory](https://code.claude.com/docs/en/memory) — the CLAUDE.md, rules, and auto-memory system.
- [Anthropic · Prompt caching](https://platform.claude.com/docs/en/docs/build-with-claude/prompt-caching) — TTL, pricing, and what gets cached.

---

## Related deep dives

- [`why-context-matters.md`](./why-context-matters.md) — the three facts about LLM runtime that make this lifecycle load-bearing in the first place.
- [`per-turn-cost-math.md`](./per-turn-cost-math.md) — what cache hits, cache writes, and mid-session invalidation actually cost per turn.
