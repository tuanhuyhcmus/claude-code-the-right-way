# The real per-turn cost of a Claude Code session

*Why adding a skill is never free — and how to calculate what a long session actually costs.*

This doc does the math behind the claim in the main README: **every primitive you load has a recurring per-turn cost, not a one-time install cost.** If you have ever wondered why a long session with many skills "feels expensive," this explains it quantitatively.

Numbers here use **Claude Opus 4.7** pricing as published by Anthropic at the time of writing. Rates change; verify against the [official pricing page](https://platform.claude.com/docs/en/about-claude/pricing) before treating these as binding.

---

## Step 1 — The four prices you need to know

Anthropic charges different rates for different kinds of input tokens. For Opus 4.7:

| Token kind                 | Price per 1M tokens | Multiplier vs. base |
| :------------------------- | :------------------ | :------------------ |
| **Regular input**          | $5.00               | 1.0×                |
| **Cache write (5-minute)** | $6.25               | 1.25×               |
| **Cache write (1-hour)**   | $10.00              | 2.0×                |
| **Cache read**             | $0.50               | 0.1×                |
| **Output**                 | $25.00              | 5.0× (for context)  |

Two consequences that matter:

- A **cache hit is an order of magnitude cheaper** than a cache miss (0.1× vs. 1.0×).
- A **cache write costs 25% extra** on top of the regular input price. You pay a one-time premium to write, then read cheaply for the cache's lifetime.

The economics of Claude Code hinge on keeping expensive content cached, not on reducing content size. **Size matters, but *stability* matters more.**

---

## Step 2 — What gets cached, and what invalidates it

Every request Claude Code sends to Anthropic looks structurally like:

```
<tools>           ← rarely changes
<system prompt>   ← rarely changes
<CLAUDE.md>       ← rarely changes
<rules>           ← rarely changes (unless path-scoped fires)
<skill descriptions>  ← rarely changes
<MEMORY.md>       ← rarely changes
<message history> ← grows every turn
<your new prompt>
```

Anthropic's prompt cache works on **prefix matching**: the cache remembers the longest exact-match prefix from the start of your request. Everything up to the first point of difference is a cache hit.

This means:

- **Stable prefix, growing suffix:** If tools, system prompt, CLAUDE.md, rules, skills, memory, and earlier message history are identical to the previous request, all of those are served as a cache hit. Only the new prompt (and its growing suffix of turns) costs full-rate input.
- **Prefix changes = cache miss for everything downstream.** If you edit `CLAUDE.md` mid-session, the cache prefix splits at the `<CLAUDE.md>` boundary. Every byte after that point (rules, skills, memory, full message history) has to be re-written to the cache on the next turn. **An edit near the top of the prefix is expensive; an edit near the bottom is cheap.**
- **Idle too long = cache expires.** The default cache TTL is 5 minutes. If no request reuses the cache within 5 minutes, the entry is evicted and the next request rewrites from scratch. The 1-hour tier costs 2× to write but lets you idle longer without re-paying.

### What invalidates the cache

| Event                                            | Invalidates from…                       | Net cost                          |
| :----------------------------------------------- | :-------------------------------------- | :-------------------------------- |
| Normal turn (same prefix, new prompt)            | Nothing — full hit                      | Cheap                             |
| 5+ minutes idle (default TTL)                    | Entire prefix                           | Full rewrite next turn            |
| Edit `CLAUDE.md` or an always-loaded rule        | From `<CLAUDE.md>` onward               | Rewrite prefix + history          |
| Add / remove / edit a skill description          | From `<skill descriptions>` onward      | Rewrite skills + memory + history |
| Invoke a new skill mid-session                   | From the skill injection point onward   | Rewrite suffix                    |
| Edit `MEMORY.md` (auto-memory append)            | From `<MEMORY.md>` onward               | Rewrite memory + history          |
| Path-scoped rule fires (new `paths:` match)      | From the rule's injection point onward  | Rewrite suffix                    |

The practical rule: **the earlier in the prefix a change lands, the more tokens the next turn has to rewrite.**

---

## Step 3 — The cost of a single long session, worked example

Assume a realistic setup:

- Startup prefix (tools + system + CLAUDE.md + rules + memory + skill descriptions): **30,000 tokens**
- Per-turn new content (your prompt + Claude's response + tool results): **~2,000 tokens average**
- Session length: **100 turns**, spaced such that cache stays warm (< 5 min between turns)

### Scenario A — ideal, cache-warm throughout

- Turn 1: prefix **cache write** (30k tokens × $6.25/MTok) = **$0.188**
- Turns 2–100: prefix **cache read** (30k × $0.50/MTok) × 99 = **$1.485**
- Per-turn new input (2k × $5/MTok) × 100 = **$1.000**
- **Total input-side cost: ~$2.67**

Most of the cost is the 2k-token suffix per turn, *not* the 30k-token prefix. That is the power of prompt caching: the 30k stays cheap because it is read, not re-written.

### Scenario B — edit `CLAUDE.md` at turn 50

Between turn 49 and turn 50, you add a line to `CLAUDE.md`. The prefix up to the first cache breakpoint is now different.

- Turns 1–49: as in scenario A.
- Turn 50: prefix **cache miss**, re-written. That is 30k + 49×2k accumulated history ≈ **128k tokens** at cache-write rate (128k × $6.25/MTok) = **$0.800**. One edit just cost you about 80 cents in extra cache-write fees.
- Turns 51–100: resume cache-warm behavior, reading the *new* prefix.

**Net penalty for one mid-session `CLAUDE.md` edit: ~$0.80.**

Multiply by a team of 10 engineers, each editing `CLAUDE.md` casually a few times a day during exploratory sessions, and this becomes real money — but more importantly, it is *invisible* money. Nobody sees it. The billing line just says "API usage."

### Scenario C — 10-minute coffee break in the middle

Between turn 50 and turn 51, you idle 10 minutes. Default 5-minute cache is evicted.

- Turn 51: prefix **cache miss** on the full current prefix + 50 turns of accumulated history — roughly **130k tokens** re-written. Cost: ~$0.81.
- You just paid the equivalent of a `CLAUDE.md` edit, to drink coffee.

This is why **the 1-hour cache tier exists**. At 2× the base write price, it is more expensive per turn to write, but pays for itself after a single cache miss you would otherwise have had.

### Break-even for the 1-hour cache

Let P be your prefix size in tokens, and let T be the expected number of cache misses you would incur under the 5-minute policy during a session.

- Default policy: T × P × $6.25/MTok
- 1-hour policy: 1 × P × $10.00/MTok + (expected refreshes within the hour) × P × $0.50/MTok

The 1-hour tier starts paying for itself around **T ≥ 2** — that is, if you expect to idle past 5 minutes twice in a session, you come out ahead. For long-running interactive sessions with human-speed pauses, that is almost always the case.

---

## Step 4 — The cost of each primitive, amortized over a session

With the math above, we can price each architectural choice.

Assume a 100-turn cache-warm session. Adding content X to the always-loaded prefix costs, per session:

- Cache-write cost on first turn (if X was added fresh): `|X| × $6.25/MTok`
- Cache-read cost on remaining 99 turns: `99 × |X| × $0.50/MTok`
- Per-turn marginal cost: **`|X| × $0.55/MTok`** (approximately, for a cache-warm long session)

Applied to realistic primitive sizes:

| Primitive                                                   | Typical size        | Cost per 100-turn session |
| :---------------------------------------------------------- | :------------------ | :------------------------ |
| One skill description                                       | ~100 tokens         | ~$0.006 (half a cent)     |
| "N skills in one framework" × 50 skills                     | ~5,000 tokens       | ~$0.28                    |
| 400-line bloated `CLAUDE.md`                                | ~5,000 tokens       | ~$0.28                    |
| A `.claude/rules/integration-test-common.md` at 3,200 tokens | ~3,200 tokens       | ~$0.18                    |
| An MCP server with 50 tool schemas                          | ~10,000 tokens      | ~$0.55                    |

Per session these are small numbers. Per team per day, they are not. A 20-person team with 10 sessions per day each: **$0.55 × 50 tools × 20 people × 10 sessions = $55 per day** just to carry one bloated MCP server across every session. That is $20k/year before anyone invokes a tool.

Even a single bloated `CLAUDE.md` sitting in the prefix of every session is a recurring tax that is almost always invisible to the people paying it.

---

## Step 5 — What this means for repo design

The math reframes some choices that look free at write-time:

- **Adding a skill is not free.** Every description you keep in the always-loaded tier costs tokens on every turn of every session, forever, whether the skill is invoked or not. `disable-model-invocation: true` genuinely removes that tax — the description is never added to the startup prompt.
- **Bloating `CLAUDE.md` is expensive in two ways.** First, the size itself costs per turn (see the table). Second, editing a big `CLAUDE.md` busts the cache for the entire downstream prefix + message history, incurring a cache-write penalty proportional to how long the session has been running.
- **Path-scoped rules are financially correct** where they apply. A rule that only loads when Claude touches `src/billing/**` costs zero tokens on turns that never touch that directory. The tradeoff: path-scoped rules are lost after `/compact` and reload on next match, which is cheap if matches are rare.
- **Subagents genuinely save money on heavy reads.** A subagent's context window is billed separately from yours. If you delegate a 50k-token research task to a subagent, the parent conversation pays for a ~1k-token summary on return, not for the 50k of file reads.
- **The 1-hour cache tier is almost always worth it** for interactive sessions. If you are in a session with any human-paced pauses, you will come out ahead.

None of this is a reason to stop using primitives. It is a reason to **budget them** — treat adding a skill or rule like adding a dependency to a hot loop. Only the ones that justify their per-turn cost should stay in the always-loaded tier. The rest belong behind `disable-model-invocation`, `paths:` scoping, or manual `/commands`.

---

## Sources

- [Anthropic Prompt Caching — Pricing and TTL](https://platform.claude.com/docs/en/docs/build-with-claude/prompt-caching)
- [Claude Code — Context window simulation](https://code.claude.com/docs/en/context-window)
- [Claude Code — Skills](https://code.claude.com/docs/en/skills)
- [Anthropic — Pricing](https://platform.claude.com/docs/en/about-claude/pricing)
