# Claude Code, the right way

> **Languages:** **English** · [Tiếng Việt](./README.vi.md)

**Target of this guide: help you organize your Claude Code setup — `CLAUDE.md`, rules, skills, hooks, memory, subagents — so they work *with* how the model actually processes your requests, not against it. You finish with a mental model for deciding where any piece of project knowledge should live.**

Most Claude Code users treat context passively: start a session, work, and let file reads, tool output, skills, and conversation history accumulate on their own. That holds on short sessions with frontier models. It breaks on long sessions, on weaker models, and on setups where you have stacked several extensions without anyone measuring the total.

**The mindset this guide argues for: treat every skill, rule, hook, agent, and MCP server in your `.claude/` as an *active tool for managing Claude's attention* — not a passive convenience collection.** Every item you add either helps the model focus or feeds it noise. The mechanics below tell you which is which, and the rest of the guide turns that into concrete placement decisions.

Concretely, the mindset looks like:

- Before you **add** a skill, ask what it costs in context and what the model loses without it.
- Before you **always-load** a rule, ask whether path-scoping with `paths:` would be cheaper.
- When Claude keeps forgetting a MUST-NOT across turns, **promote** it to a hook instead of re-emphasising the prose.
- When a task requires reading thousands of lines of source, **push** the read into a subagent so the main session stays clean.
- When a session starts feeling slower or worse, the first thing to check is not the model — it is `/context`, and what has filled up the window.

This is the highest-leverage habit in Claude Code. It is the single thing that most cleanly separates setups that feel sharp from setups that feel sluggish, on the same model and the same repo.

This guide is opinionated. It is closer to a *"here is what I think most people get wrong"* essay than a neutral reference. If you want neutral, the [Claude Code docs](https://docs.claude.com/en/docs/claude-code) are better.

> Based on daily use across a mid-size Go monorepo. Stack-specific patterns will differ on TypeScript / Python / Rust / Elixir setups — treat everything here as a starting point, not universal law. Sample size: one.

<details>
<summary><strong>First, a quick refresher</strong> — click to expand if you want a reminder of what <code>CLAUDE.md</code>, rules, skills, hooks, memory, and subagents actually are. Skip if you already know them.</summary>

<br/>

**`CLAUDE.md`** — a markdown file at the repo (or subdirectory) root. Claude reads it every turn, so it's the right place for behavioral guidelines that apply to all tasks ("prefer editing over creating files", "don't add comments unless asked"). [Official docs](https://docs.claude.com/en/docs/claude-code/memory).

**Rules** (`.claude/rules/*.md`) — markdown files under `.claude/rules/` that Claude Code auto-discovers recursively and loads with the same priority as `.claude/CLAUDE.md`. Use them to split project invariants ("MUST / MUST NOT" statements) out of a bloated `CLAUDE.md`. A YAML `paths:` frontmatter scopes a rule to specific file patterns so it only loads when Claude touches matching files — great for reducing context noise. User-level rules live at `~/.claude/rules/`. [Official docs](https://code.claude.com/docs/en/memory#organize-rules-with-claude/rules/).

**Skills** (`.claude/skills/<name>/SKILL.md`) — named, trigger-based workflows Claude can invoke. Each skill has a description telling Claude when to use it, plus procedural steps inside. Great for "when the user says X, do Y₁ → Y₂ → Y₃". [Official docs](https://docs.claude.com/en/docs/claude-code/skills).

**Hooks** (`.claude/hooks/*.sh` + `settings.json`) — shell scripts the Claude Code harness runs automatically at lifecycle events (pre-tool, post-tool, user-prompt-submit, etc.). Non-zero exit can block an action. This is your *machine enforcement* layer — the only thing that stops Claude from violating a rule even when the rule is loaded in context. [Official docs](https://docs.claude.com/en/docs/claude-code/hooks).

**Memory** (`~/.claude/projects/<project>/memory/*.md`) — user-local, persists across sessions. Claude writes here automatically to remember facts about you, your project, and your preferences. *Not* for things the code or `git log` can answer. [Official docs](https://docs.claude.com/en/docs/claude-code/memory).

**Subagents** (invoked via the `Agent` tool) — a fresh Claude instance with its own context window, spawned to handle a scoped task. The parent only sees the final summary. Use them to isolate heavy reads (thousands of lines) or parallel independent work. [Official docs](https://docs.claude.com/en/docs/claude-code/sub-agents).

</details>

---

## How context works under the hood

To apply the mindset well, you need three mechanical facts (plus two nuances) about how LLMs and Claude Code actually process a request. Every placement decision later in this guide is a corollary of them. Skip to [the primitives](#the-primitives-at-a-glance) if you already know the mechanics; read on if you want the underlying reasons.

### Fact 1 — The context window is finite, and shared

A model can only "see" a fixed number of tokens at once — its context window. Claude Opus 4.7 tops out at 1M tokens. That sounds enormous. It is not, because the budget is **shared**, not dedicated to your question.

Everything below competes for the same 1M tokens on every single turn:

- Claude Code's own system prompt (several thousand tokens just to define its behavior)
- Every tool's JSON schema — built-in tools, MCP tools, agent definitions (often 10–20k tokens for a typical setup; MCP servers are usually the largest contributor)
- All `CLAUDE.md` files loaded by directory walk-up
- All always-loaded `.claude/rules/*.md`
- Every model-invocable skill's description (not the body — just metadata)
- `MEMORY.md` (up to 25KB)
- The entire prior conversation: your messages, Claude's replies, and every tool result
- File contents Claude has read, grep output, bash output
- The task output Claude is building right now

A single `Read` of a 2000-line source file can eat 30–50k tokens. Ten such reads in a long debugging session and you have burned 5% of the window before doing any actual work. Ask Claude to dump a 500-line file three times — 150k tokens gone, competing against the skill description you need Claude to trigger on.

> **Don't trust the numbers above — measure your own.** Claude Code exposes the `/context` command, which breaks down current session usage by category (system prompt, tools, CLAUDE.md, conversation, files, etc.). Run it on a few real sessions before optimising anything; the mix varies wildly by MCP setup, repo size, and conversation style.

"1M tokens" is the ceiling, not the workspace. The usable workspace shrinks every minute of a session.

### Fact 2 — Every request re-sends the entire conversation

LLM inference is **stateless**. The model remembers nothing between requests. Each time Claude responds, the client re-sends the full prior conversation, all loaded CLAUDE.md files, all rules, all skill descriptions, all tool definitions — everything, from turn 1.

A practical consequence: if you add a 500-line instruction to `CLAUDE.md`, those 500 lines are included in every request for the rest of the session. The "upfront tax" is not a one-time cost — it is a per-turn cost multiplied by every turn you ever have.

#### Nuance 1 — prompt caching softens the *cost*, not the *context tax*

Anthropic's API supports [prompt caching](https://docs.claude.com/en/docs/build-with-claude/prompt-caching). The key facts most guides skip:

- **Cache reads are ~10% of input price.** Cheap.
- **Cache writes cost ~125% of input price.** Turn 1 with a 1,000-line `CLAUDE.md` is *more* expensive than no caching at all. The discount only pays back after several subsequent turns amortise the write.
- **Default TTL is 5 minutes** (1-hour tier available in beta). Leave a session idle over lunch and the cache evicts — next turn re-writes and re-pays the 125% cost.
- **Caching is per breakpoint.** The API accepts up to 4 `cache_control` markers. Where Claude Code places them (system prompt, tool definitions, CLAUDE.md, conversation) determines which parts actually cache. Touch anything before a breakpoint and everything from that breakpoint onward has to be re-sent.

So *"you pay every turn"* is accurate for the **context budget** (tokens still occupy the window and still compete for attention) but overstated for **dollar cost and latency** once a session is warm. It is under-stated for short sessions and after every idle gap.

The implication for optimising `CLAUDE.md` and rules is not simply *"keep them small."* It is:

- **Stable-within-session beats small.** A 1,000-line `CLAUDE.md` that never changes across a continuous session caches well after the write-cost turn. Edit it mid-session → cache invalidates → re-pay the 125%.
- **Keep volatile content out of the upfront tax.** Content that changes frequently (today's debugging state, current task notes) belongs in conversation or ephemeral memory, not in `CLAUDE.md` where a single edit busts the cache for every turn after.
- **Short sessions punish bloat.** If most of your sessions are <5 turns, the cache-write cost dominates and the "stability" advice flips: small beats stable-but-large.
- **Context budget is still finite.** Caching makes tokens cheaper and faster; it does not make them smaller. Attention dilution (Fact 3 below) applies regardless of whether a token came from cache or fresh.

#### Nuance 2 — `/compact` rewrites conversation history mid-session

Claude Code auto-compacts when context fills. Post-compaction, the client does not resend the raw conversation from turn 1 — it sends a **summary** plus the most recent turns. Specifically:

- Project-root `CLAUDE.md` **is re-injected** from disk (so root-level instructions survive).
- Nested `CLAUDE.md` files in subdirectories **are NOT re-injected**; they reload only when Claude next touches a file in that subtree.
- Skill bodies that were invoked **are re-attached** with a token budget (most recent first; older skill invocations may be dropped).
- The rest of the conversation becomes a model-generated summary.

So *"every request re-sends the full conversation"* is only true up to the first compaction event. Long sessions (>2 hours) typically cross several. This matters because:

- A CLAUDE.md edit mid-session is worse than stated above: it not only busts the cache, it is the version that gets re-injected after every subsequent compaction, potentially overwriting earlier context that Claude had developed.
- Nested `CLAUDE.md` is more fragile than root: one `/compact` and it is gone until Claude touches that subdirectory again.
- Skills invoked early may be silently dropped after enough compactions. If a critical skill needs to persist, re-invoke it.

### Fact 3 — Bigger context does not mean better focus

Self-attention — the mechanism the model uses to decide which tokens matter — gives every token a weight, and **those weights must sum to a fixed total**. That means every additional irrelevant token reduces the weight available for the relevant ones. More context makes the signal thinner, not sharper.

Combined with the empirically-measured "lost in the middle" effect (info buried between long prefixes and long suffixes gets weighted less), a prompt twice as long can be *less* effective than a prompt half the size, if the extra content is noise. More context is not a free upgrade.

A practical corollary, important enough to pull out: **the optimal number of skills (or rules, or MCP tools) is not "as many as possible"; it is as few as possible for your target model's attention budget.** A setup that works beautifully on Opus 4.x can break on a self-hosted 30B-parameter coding model not because the smaller model is "dumb," but because every additional description, rule, and tool schema dilutes the attention weight available for the one Claude actually needs.

> **Deep dive:** [`docs/llm-mechanics/self-attention.md`](./docs/llm-mechanics/self-attention.md) — an analogy-driven explainer for how self-attention actually works, why 1M tokens is a ceiling not a superpower, and what it means for writing Claude Code prompts. Read this if you have not read transformer papers and want the intuition without the math.

### What this means for placement

Most of the core primitives this guide focuses on exist to answer the same question: *how do we feed the model exactly what it needs, exactly when it needs it, without paying for it on every turn we don't?* (MCP servers, plugins, and other ecosystem primitives exist for different reasons — external integrations, distribution — and are mentioned briefly elsewhere.)

Three broad categories, driven by the mechanics above:

- **Pay every turn**: `CLAUDE.md`, always-loaded rules, skill **descriptions**. Bloat them and every response gets slower, more expensive, and less focused.
- **Pay only when relevant**: skill **bodies**, path-scoped rules, memory topic files, subagents. The escape hatch for heavy content.
- **Pay nothing at the model level**: hooks and `permissions.deny`. Enforcement that lives outside the context window entirely.

Most of the placement arguments in the rest of this guide reduce to a corollary of the three facts above plus the two nuances. The real question is not *whether* to think in these categories, but *which* knobs you personally should touch for your codebase and session patterns.

---

## The primitives (at a glance)

| Primitive | Nature | Lifetime | Scope | Activated by |
|---|---|---|---|---|
| `CLAUDE.md` | Behavioral guideline (prose) | Committed | Repo / dir | Claude reads every turn (consumption, not enforcement) |
| `.claude/rules/*.md` | Declarative invariant (MUST / MUST NOT) | Committed | Repo | Loaded with CLAUDE.md; path-scoped variants load on matching file reads |
| `.claude/skills/*/SKILL.md` | Procedural workflow (how-to) | Committed | Repo / user | Claude invokes implicitly by description match, or user via `/name` |
| `.claude/hooks/*.sh` | Machine enforcement | Committed | Repo | Harness at pre/post tool lifecycle events (real enforcement) |
| `memory/*.md` | Ephemeral user / project state | User-local | Cross-session | `MEMORY.md` auto-loaded; topic files on demand |
| Subagent | Context isolation | Per-invocation | Task-scoped | `Agent` tool call by parent |

Each primitive answers a different question. In my experience, mixing them up is the most frequent mis-placement.

> **This table is not the full set.** Things this guide does not cover in depth but which live in the same ecosystem: **MCP servers** (often the single largest source of upfront tool-schema tokens in a session), **plugins** (official mechanism for bundling skills + hooks + agents), **settings** (`settings.json`, `settings.local.json`, managed settings — govern permissions, hooks, environment), **slash commands** (merged into skills in late 2025; a file at `.claude/commands/x.md` and a skill at `.claude/skills/x/SKILL.md` both produce `/x`), and **output styles**. Everything in this guide applies to them too, but exhaustive coverage would double the length.

---

## Two costs, one enforcement

Placement decisions get much easier once you internalize that these primitives live on **three different axes**, not one. A lot of docs treat them as interchangeable tools — they are not.

### Axis 1 — Context tax every turn

These load into Claude's context at the start of every conversation, whether you use them or not:

- `CLAUDE.md` — full content
- `.claude/rules/*.md` without a `paths:` frontmatter — full content
- Skill **descriptions** (the YAML frontmatter) — metadata only, but loaded for every *model-invocable* skill (skills with `disable-model-invocation: true` are excluded from this tax — see [Two triggers](#two-triggers-explicit-vs-implicit) below)
- Subagent **descriptions** — same
- `MEMORY.md` — first 200 lines or 25KB, whichever comes first

Budget accordingly. A repo with 50 skills × 20-line descriptions is ~1000 lines of upfront tax before you type anything.

### Axis 2 — Prompt enrichment on-demand

These enter context only when Claude needs them. But "on-demand" is not synonymous with "ephemeral" — their *post-load lifecycles* differ, and the differences matter for context budgeting:

| Primitive                         | Loads when                                  | Lifecycle after loading                                                                             |
| :-------------------------------- | :------------------------------------------ | :-------------------------------------------------------------------------------------------------- |
| **Skill body**                    | Claude invokes the skill                    | Enters the transcript as a single message; [docs confirm](https://code.claude.com/docs/en/skills#skill-content-lifecycle) it stays for the rest of the session and is re-attached after `/compact` with a token budget |
| **Subagent execution context**    | Parent calls the `Agent` tool               | Fully isolated in a fresh context window; parent only sees the final summary                        |
| **Path-scoped rule** (`paths:`)   | Claude reads a file matching the pattern    | Injected when the matching file is read; stays in transcript as part of conversation history (not re-fetched on later turns) |
| **Memory topic file**             | Claude opens it via the `Read` tool         | Behaves like any file read — content lands in the transcript and stays until `/compact` summarises it away |
| **Nested `CLAUDE.md`**            | Claude touches a file in its subdirectory   | Accumulates as you descend deeper; **not re-injected after `/compact`** (unlike project-root `CLAUDE.md`) |

This is where heavy, task-specific content belongs — free from the upfront "every turn" tax. Two traps worth naming:

- **"On-demand" is not "ephemeral."** Skill bodies and file reads stay in transcript after loading. Invoking ten skills in one session accumulates all ten bodies for the rest of the session.
- **Lifecycle claims in this table reflect observed behaviour and public docs at the time of writing.** Claude Code's handling of path-scoped rule re-injection and nested CLAUDE.md after compaction has evolved across releases. Before budgeting heavily around these, run a small test in your current version.

### Axis 3 — Outside the model entirely

Hooks are different in kind. They run in the Claude Code harness (Node.js client), not inside the model:

- No context tokens consumed
- Deterministic: shell exit code decides block / pass; the model cannot override
- Cannot be "forgotten" mid-conversation
- Limited to what a shell command can actually verify

Hooks are one of two layers that turn "MUST NOT" from a hope into enforcement. The other is `permissions.deny` in `settings.json`:

```json
{
  "permissions": {
    "deny": ["Bash(rm -rf *)", "Read(./.env)"]
  }
}
```

Both run in the harness, both are deterministic, both bypass the model entirely. Hooks are more flexible (arbitrary shell logic, can inspect tool inputs, can inject messages back); `permissions.deny` is simpler and declarative. Use both. Every other primitive depends on Claude reading it, understanding it, and choosing to follow it.

---

### The trust-boundary split in skills and subagents

Descriptions and bodies live on different axes — and they can say different things.

|                 | When Claude sees it        | What it's for                      |
| --------------- | -------------------------- | ---------------------------------- |
| **Description** | Every turn (Axis 1)        | Routing: *"should I invoke this?"* |
| **Body**        | Only after invoking (Axis 2) | Execution: *"what do I do now?"*   |

Claude routes based on the description alone. The body does not get to "disagree" until after the decision to invoke has already been made. This split is both a design feature and a footgun:

- **Feature.** 50 skills do not cost 50 × body-length tokens every turn. Only their descriptions do. That is what makes large skill libraries viable.
- **Drift footgun.** A description that promises X but a body that does Y causes Claude to invoke in the wrong context — and you only discover the mismatch after the skill has already run.
- **Supply-chain risk — real, but not unmitigated.** A skill shared from an external repo can pair a benign-looking description ("code formatter") with an unexpected body. Defences exist in layers: Claude Code prompts for approval on external imports, PreToolUse hooks fire on tool calls inside the body, `permissions.deny` blocks disallowed tools, and the user sees each tool call in the UI before approval (in default mode). Realistic threat = careful review is required for third-party skills, not that any install auto-pwns you. But the trust-boundary split is still real: a description you scanned is not a substitute for a body you scanned.

> **Description is the routing contract. Body is the execution contract. They can diverge — and that is both a feature and a vulnerability. When adopting a third-party skill, read the body, not just the description.**

---

### Two triggers: explicit vs implicit

Skills and subagents can be invoked in two different ways, and the choice affects reliability, safety, and context cost. Getting this right is usually more important than any individual skill's content.

There are two ways to invoke a skill or subagent:

1. **Explicitly** — the user types `/skill-name`, or the parent agent calls the `Agent` tool with a specific subagent type. Deterministic. The user (or the parent model) is in control of timing.

2. **Implicitly** — Claude reads the description every turn, matches it against the current user prompt, and decides on its own whether to invoke. This is the "it just knew what I wanted" mode. It is also the mode where a lot of behavior that looks like intelligence is actually careful description engineering.

Claude Code exposes explicit knobs for both in skill frontmatter:

| Frontmatter                         | Invocation                                        | Use for                                                                                                                                  |
| :---------------------------------- | :------------------------------------------------ | :--------------------------------------------------------------------------------------------------------------------------------------- |
| (default)                           | Both `/command` and auto-trigger                  | Most skills                                                                                                                              |
| `disable-model-invocation: true`    | Only `/command`. Claude cannot auto-trigger.      | Anything with side effects: `/commit`, `/deploy`, `/send-slack-message`. You do not want Claude deciding *when* to deploy based on vibes. |
| `user-invocable: false`             | Only Claude auto-triggers. Hidden from `/` menu.  | Background knowledge (`legacy-system-context`, `api-conventions`) — not actionable as a command, but Claude should pull it in when relevant. |
| `paths: src/billing/**`             | Claude auto-triggers only when touching matching files | Domain-specific skills that should not compete for attention outside their area                                                          |

Subagents have a similar split: the parent model picks which subagent type to spawn based on the `Agent` tool description, but you can also constrain which agents are available in a given context.

**Why this matters:**

- **Implicit trigger depends on description engineering.** When a skill seems to "just know" when to activate, it is because someone tuned the description, the `when_to_use` hints, and the path scope so the model reliably auto-invokes on the right prompts. That is real work — and it is failure-prone. A description has to win the attention competition against every other skill in the budget.
- **Explicit trigger is a safety lever, especially on weaker models.** Auto-trigger assumes the model is smart enough to match a prompt to a description correctly. Frontier models (Opus 4.x, Sonnet 4.6, and Haiku 4.5 in practice) auto-trigger adequately for most well-written skill descriptions. The gap shows up sharply on smaller open-source coding models in the 30B-parameter class — DeepSeek Coder, Qwen Coder, CodeLlama and similar. These models are increasingly capable of *executing* skill bodies once invoked, but their *routing* accuracy (matching a user prompt to the right skill description, out of 30+ candidates, on every turn) drops visibly. Irrelevant skills fire, relevant ones get skipped. If you target OSS models for self-hosted setups, forcing `/command` invocation is the cheapest way to sidestep this.
- **Actions with side effects should almost always be explicit.** If a skill deploys, sends messages, mutates shared state, or spends money, you do not want "Claude decided to" in the post-mortem. `disable-model-invocation: true` is cheap insurance.
- **Pure knowledge injection is fine implicit.** A skill that says *"when touching billing code, here are the invariants"* is ideal auto-trigger material — scope it with `paths:`, hide it from the menu with `user-invocable: false`. Users should not have to memorize `/billing-context`.

---

## Appendix — decision tree: "where does this knowledge live?"

The mental model above is the actual product of this guide. This tree is an appendix that applies the model to the most common placement question. Use it to pick the **primary placement**, then consider the supplementary-placement note below — real knowledge often belongs in more than one primitive.

```
Is it derivable from `git log` or reading the code with low cost?
  └── YES → store NOTHING. Don't pollute context.

Is it a MUST / MUST NOT that's always true, regardless of task?
  ├── Machine-checkable (grep, lint)? → rule + hook (both, not either)
  └── Human-enforced only?            → rule

Is it a "when user says X, do Y₁ → Y₂ → Y₃" workflow?
  └── YES → skill

Does answering it require reading enough source that it will noticeably
pollute the main context (rule of thumb: several thousand lines of reads)?
  └── YES → subagent (isolate from main context)

Is it a fact about the user or project that changes over time?
  └── YES → memory

Is it style / tone / philosophy spanning the whole repo?
  └── YES → CLAUDE.md
```

**Supplementary placements.** The same piece of knowledge often earns a home in more than one primitive — not because you are duplicating, but because each primitive addresses a different failure mode:

- *"All outbound HTTP must use our `httpclient.Do` wrapper"* → **rule** (declarative invariant Claude reads) + **hook** (grep denies raw `http.Get(` in PreToolUse) + optional **skill** (`/migrate-http-call` to refactor existing call sites).
- *"Billing code invariants"* → path-scoped **rule** (only loaded when touching `src/billing/**`) + optional **skill** (`/billing-review`, `user-invocable: false` so Claude auto-loads it in billing context).
- *"Preferred commit message style"* → **CLAUDE.md** (one-line convention) + explicit **skill** `/commit` (procedure for scripted commits).

The rule of thumb: if something is *declarative + enforceable*, do both rule and hook. If something is *knowledge + procedure*, consider both rule/CLAUDE.md and skill. The decision tree picks the **most important** placement; it does not say "only place it there."

If nothing matches, you probably don't need to persist it.

---

## Start small, curate deliberately

Two ideas worth leaving with.

**1. Organise for the model's weaknesses, not around them.**
A well-curated `.claude/` is not just convenient — it is a *context-management strategy*. Every item you add either helps the model focus or fights it. The mechanics earlier in this guide are not abstract physics; they are the budget constraints your curation operates under. Use them as a checklist:

- **Fewer, sharper skill descriptions** → less attention dilution (Fact 3). The target skill wins its Q/K match more often. Ten well-tuned descriptions usually beat thirty generic ones.
- **Path-scoped rules over always-loaded ones** (`paths: src/billing/**`) → rules enter context only when relevant, freeing the upfront budget for things the model needs globally.
- **Subagents for heavy reads** → 50k tokens of file contents that would have polluted your main session stay in an isolated window, and you get back a summary instead of the raw dump.
- **Hooks for MUST rules** → zero context cost, zero dependence on Claude remembering across turns, and zero attention budget spent on enforcement.
- **`permissions.deny` for hard blocks** → same deal as hooks for prohibitions that need no logic.
- **CLAUDE.md stable within a session** → cache stays warm, cost and latency drop (Caching nuance).
- **Prune MCP servers you do not actually use** → often the single largest token win. 5–10k tokens back on every turn, for the rest of the session.

The question to ask each item in your `.claude/`: *what does this cost me in tokens, and what would I lose by not having it?* If you cannot answer the second half, it is dead weight. Delete it.

**2. Start small. Cherry-pick before authoring.**
You do not need a name, a repo, or a master plan. You just need a `.claude/` directory with the handful of skills, rules, hooks, and CLAUDE.md lines that fit your codebase, model, and session patterns.

The best way to build that is not writing from scratch. It is assembling:

- Browse the skills and agents shared by others (popular bundles, blog posts, colleagues' setups). Lift the ones that solve a problem you actually have, adapt them to your naming and conventions, drop the ones that do not fit. You own your copy from that point on.
- Add things one at a time as needs arise, not upfront:
  - Add one **rule** for an invariant you currently catch in code review.
  - Add one **skill** for a procedure you type the same way every time (`/commit`, `/review-pr`, whatever).
  - Add one **hook** when you notice Claude keeps violating the same rule and you are tired of reminding it.
- Every time you add something, run `/context` and ask: *is it worth what it costs?*

In six months you will have a setup nobody else has — and nobody else needs. That is the goal: not a published framework, but a working environment that fits *your* codebase and *your* model's attention budget. **You already know your codebase better than any config author on GitHub ever will. The mindset above is what lets you act on that.**

---

## Scope boundaries

- Not an "awesome-claude" list. Those exist.
- Not a tutorial on installing Claude Code. See [official docs](https://docs.claude.com/en/docs/claude-code).
- Not a comprehensive prompt-engineering handbook. It touches prompt-engineering-adjacent topics (attention dilution, description craft, caching) where they are load-bearing for the argument; general prompt craft is out of scope.
- Not a ranking of Claude models. Specific model behaviours are mentioned where they shift the argument, but volatile enough that you should re-verify rather than trust this guide's snapshot.

Issues and disagreements welcome. If you think a framework implication here is wrong, open an issue with a concrete counter-example — this guide is more useful as a disputed claim than as unchallenged doctrine.

## License

MIT — see [LICENSE](./LICENSE).
