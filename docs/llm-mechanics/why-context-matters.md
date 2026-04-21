# Why context matters: three facts about LLM runtime

*The mechanical reasons every Claude Code primitive exists.*

This doc is the foundation of the main README's reasoning. Every placement decision in this repo — what goes in `CLAUDE.md`, what goes in rules, what goes in skills, what stays outside the model in hooks — flows from three facts about how LLMs actually run at inference time. If you already know how LLMs work at runtime, you can skip this. If you do not, start here before reading anything else.

---

## Fact 1 — The context window is finite, and shared

A model can only "see" a fixed number of tokens at once — its context window. Claude Opus 4.7 tops out at 1M tokens. That sounds enormous. It is not, because the budget is **shared**, not dedicated to your question.

Everything below competes for the same 1M tokens on every single turn:

- Claude Code's own system prompt (several thousand tokens just to define its behavior)
- Every tool's JSON schema — built-in tools, MCP tools, agent definitions (easily 10–20k tokens for a typical setup)
- All `CLAUDE.md` files loaded by directory walk-up
- All always-loaded `.claude/rules/*.md`
- Every skill's description (not the body — just metadata — but for *every* skill)
- `MEMORY.md` (up to 25KB)
- The entire prior conversation: your messages, Claude's replies, and every tool result
- File contents Claude has read, grep output, bash output
- The task output Claude is building right now

A single `Read` of a 2000-line source file can eat 30–50k tokens. Ten such reads in a long debugging session and you have burned 5% of the window before doing any actual work. Ask Claude to dump a 500-line file three times — 150k tokens gone, competing against the skill description you need Claude to trigger on.

**"1M tokens" is the ceiling, not the workspace.** The usable workspace shrinks every minute of a session.

---

## Fact 2 — Every request re-sends the entire conversation

LLM inference is **stateless**. The model remembers nothing between requests. Each time Claude responds, the client re-sends the full prior conversation, all loaded `CLAUDE.md` files, all rules, all skill descriptions, all tool definitions — everything, from turn 1.

A practical consequence: if you add a 500-line instruction to `CLAUDE.md`, you pay for those 500 lines on every single response for the rest of the session. The "upfront tax" is not a one-time cost — it is a per-turn cost multiplied by every turn you ever have.

This is where prompt caching saves you. Anthropic caches the stable prefix of each request server-side for 5 minutes (or 1 hour at 2× write cost) so the re-send on turn N+1 only pays 0.1× for the prefix that matches turn N. But the cache is prefix-matched: **any change near the top of the prefix invalidates everything downstream.** See [per-turn-cost-math.md](./per-turn-cost-math.md) for what that actually costs.

---

## Fact 3 — Bigger context does not mean better focus

Self-attention — the mechanism the model uses to decide which tokens matter — gives every token a weight, and **those weights must sum to a fixed total**. That means every additional irrelevant token reduces the weight available for the relevant ones. More context makes the signal thinner, not sharper.

Combined with the empirically-measured "lost in the middle" effect (info buried between long prefixes and long suffixes gets weighted less), a prompt twice as long can be *less* effective than a prompt half the size, if the extra content is noise. **More context is not a free upgrade.**

For an analogy-driven explainer of how self-attention actually works, why 1M tokens is a ceiling rather than a superpower, and what it means for writing Claude Code prompts, see the companion deep dive: [self-attention.md](./self-attention.md).

---

## What this means for the primitives

Every primitive in Claude Code is an answer to the same question: *how do we feed the model exactly what it needs, exactly when it needs it, without paying for it on every turn we don't?*

- **`CLAUDE.md`, always-loaded rules, skill descriptions:** pay *every turn*. Bloat them and every response gets slower, more expensive, and less focused.
- **Skill bodies, path-scoped rules, memory topic files, subagents:** pay *only when relevant*. The escape hatch for heavy content.
- **Hooks:** pay *nothing* at the model level. Enforcement that lives outside the context window entirely.

Once you internalize this, the rest of the main README is just applying the principle to specific placement decisions.

---

## Related deep dives

- [`self-attention.md`](./self-attention.md) — how attention actually works, why the 1M-token window does not equal 1M effective tokens.
- [`per-turn-cost-math.md`](./per-turn-cost-math.md) — what caching, invalidation, and an edit to `CLAUDE.md` mid-session actually cost in dollars per 100 turns.
