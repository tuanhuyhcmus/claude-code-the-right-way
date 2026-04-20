# Claude Code, the right way

An opinionated guide to **organizing knowledge** in a Claude Code project.

You already know what skills, rules, memory, agents, and hooks are. The hard question is: when you learn something new about your codebase, **where does it go?** This repo is the decision tree I wish I had 6 months ago.

> Based on daily use across a mid-size Go monorepo. YMMV — treat as a starting point, not dogma.

---

## First, why does Claude Code need all this?

If you already know how LLMs work at runtime, skip to [the primitives](#the-primitives-at-a-glance). If you don't, this section is the single most important part of the repo — every placement decision below flows from three mechanical facts.

### Fact 1 — The context window is finite, and shared

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

"1M tokens" is the ceiling, not the workspace. The usable workspace shrinks every minute of a session.

### Fact 2 — Every request re-sends the entire conversation

LLM inference is **stateless**. The model remembers nothing between requests. Each time Claude responds, the client re-sends the full prior conversation, all loaded CLAUDE.md files, all rules, all skill descriptions, all tool definitions — everything, from turn 1.

A practical consequence: if you add a 500-line instruction to `CLAUDE.md`, you pay for those 500 lines on every single response for the rest of the session. The "upfront tax" is not a one-time cost — it is a per-turn cost multiplied by every turn you ever have.

### Fact 3 — Bigger context does not mean better focus

Self-attention — the mechanism the model uses to decide which tokens matter — gives every token a weight, and **those weights must sum to a fixed total**. That means every additional irrelevant token reduces the weight available for the relevant ones. More context makes the signal thinner, not sharper.

Combined with the empirically-measured "lost in the middle" effect (info buried between long prefixes and long suffixes gets weighted less), a prompt twice as long can be *less* effective than a prompt half the size, if the extra content is noise. More context is not a free upgrade.

> **Deep dive:** [`docs/llm-mechanics/self-attention.md`](./docs/llm-mechanics/self-attention.md) — an analogy-driven explainer for how self-attention actually works, why 1M tokens is a ceiling not a superpower, and what it means for writing Claude Code prompts. Read this if you have not read transformer papers and want the intuition without the math.

### What this means for the primitives

Every primitive in Claude Code is an answer to the same question: *how do we feed the model exactly what it needs, exactly when it needs it, without paying for it on every turn we don't?*

- `CLAUDE.md`, always-loaded rules, skill **descriptions**: pay *every turn*. Bloat them and every response gets slower, more expensive, and less focused.
- Skill **bodies**, path-scoped rules, memory topic files, subagents: pay *only when relevant*. The escape hatch for heavy content.
- Hooks: pay *nothing* at the model level. Enforcement that lives outside the context window entirely.

Once you internalize this, the rest of the repo is just applying the principle.

---

## Quick recap — what are these things?

<details>
<summary>Expand if you need a refresher on <code>CLAUDE.md</code>, rules, skills, hooks, memory, or subagents.</summary>

**`CLAUDE.md`** — a markdown file at the repo (or subdirectory) root. Claude reads it every turn, so it's the right place for behavioral guidelines that apply to all tasks ("prefer editing over creating files", "don't add comments unless asked"). [Official docs](https://docs.claude.com/en/docs/claude-code/memory).

**Rules** (`.claude/rules/*.md`) — markdown files under `.claude/rules/` that Claude Code auto-discovers recursively and loads with the same priority as `.claude/CLAUDE.md`. Use them to split project invariants ("MUST / MUST NOT" statements) out of a bloated `CLAUDE.md`. A YAML `paths:` frontmatter scopes a rule to specific file patterns so it only loads when Claude touches matching files — great for reducing context noise. User-level rules live at `~/.claude/rules/`. [Official docs](https://code.claude.com/docs/en/memory#organize-rules-with-claude/rules/).

**Skills** (`.claude/skills/<name>/SKILL.md`) — named, trigger-based workflows Claude can invoke. Each skill has a description telling Claude when to use it, plus procedural steps inside. Great for "when the user says X, do Y₁ → Y₂ → Y₃". [Official docs](https://docs.claude.com/en/docs/claude-code/skills).

**Hooks** (`.claude/hooks/*.sh` + `settings.json`) — shell scripts the Claude Code harness runs automatically at lifecycle events (pre-tool, post-tool, user-prompt-submit, etc.). Non-zero exit can block an action. This is your *machine enforcement* layer — the only thing that stops Claude from violating a rule even when the rule is loaded in context. [Official docs](https://docs.claude.com/en/docs/claude-code/hooks).

**Memory** (`~/.claude/projects/<project>/memory/*.md`) — user-local, persists across sessions. Claude writes here automatically to remember facts about you, your project, and your preferences. *Not* for things the code or `git log` can answer. [Official docs](https://docs.claude.com/en/docs/claude-code/memory).

**Subagents** (invoked via the `Agent` tool) — a fresh Claude instance with its own context window, spawned to handle a scoped task. The parent only sees the final summary. Use them to isolate heavy reads (thousands of lines) or parallel independent work. [Official docs](https://docs.claude.com/en/docs/claude-code/sub-agents).

</details>

---

## The primitives (at a glance)

| Primitive | Nature | Lifetime | Scope | Enforced by |
|---|---|---|---|---|
| `CLAUDE.md` | Behavioral guideline (prose) | Committed | Repo / dir | Claude reads every turn |
| `.claude/rules/*.md` | Declarative invariant (MUST / MUST NOT) | Committed | Repo | Claude + hook |
| `.claude/skills/*/SKILL.md` | Procedural workflow (how-to) | Committed | Repo / user | Claude invokes on trigger |
| `.claude/hooks/*.sh` | Machine enforcement | Committed | Repo | Harness (pre/post tool) |
| `memory/*.md` | Ephemeral user / project state | User-local | Cross-session | Claude auto-loads |
| Subagent | Context isolation | Per-invocation | Task-scoped | `Agent` tool |

Each primitive answers a different question. Mixing them up is the single most common mistake.

---

## Two costs, one enforcement

Placement decisions get much easier once you internalize that these primitives live on **three different axes**, not one. A lot of docs treat them as interchangeable tools — they are not.

### Axis 1 — Context tax every turn

These load into Claude's context at the start of every conversation, whether you use them or not:

- `CLAUDE.md` — full content
- `.claude/rules/*.md` without a `paths:` frontmatter — full content
- Skill **descriptions** (the YAML frontmatter) — metadata only, but loaded for *every* skill so Claude knows when to trigger
- Subagent **descriptions** — same
- `MEMORY.md` — first 200 lines or 25KB, whichever comes first

Budget accordingly. A repo with 50 skills × 20-line descriptions is ~1000 lines of upfront tax before you type anything.

### Axis 2 — Prompt enrichment on-demand

These only enter context when Claude decides they're relevant:

- Skill **bodies** (everything in `SKILL.md` after the frontmatter) — loaded when Claude invokes the skill
- Subagent **execution context** — isolated in a fresh window; the parent only sees the final summary
- Path-scoped rules (`paths: src/api/**`) — loaded when Claude reads a matching file
- Memory topic files (everything other than `MEMORY.md`) — loaded on demand
- Nested `CLAUDE.md` in subdirectories — loaded when Claude touches files there

This is where heavy, task-specific content belongs. It is free from the "every turn" budget.

### Axis 3 — Outside the model entirely

Hooks are different in kind. They run in the Claude Code harness (Node.js client), not inside the model:

- No context tokens consumed
- Deterministic: shell exit code decides block / pass; the model cannot override
- Cannot be "forgotten" mid-conversation
- Limited to what a shell command can actually verify

Hooks are the only layer that turns "MUST NOT" from a hope into enforcement. Every other primitive depends on Claude reading it, understanding it, and choosing to follow it.

---

### The trust-boundary split in skills and subagents

Descriptions and bodies live on different axes — and they can say different things.

|                 | When Claude sees it        | What it's for                      |
| --------------- | -------------------------- | ---------------------------------- |
| **Description** | Every turn (Axis 1)        | Routing: *"should I invoke this?"* |
| **Body**        | Only after invoking (Axis 2) | Execution: *"what do I do now?"*   |

Claude routes based on the description alone. The body does not get to "disagree" until after the decision to invoke has already been made. This split is both a design feature and a footgun:

- **Feature.** 50 skills do not cost 50 × body-length tokens every turn. Only their descriptions do. That is what makes large skill libraries viable.
- **Drift footgun.** A description that promises X but a body that does Y causes Claude to invoke in the wrong context — and discover the mismatch too late.
- **Supply-chain risk.** A skill shared from an external repo can pair a benign description ("code formatter") with a malicious body (exfiltrate secrets). Claude trusts the description to route, then executes the body inside your session. This is why Claude Code requires explicit approval for external imports.

> **Description is the routing contract. Body is the execution contract. They can diverge — and that is both a feature and a vulnerability. When reviewing a third-party skill, read the body, not just the description.**

---

### Two triggers: explicit vs implicit

If you have used one of the popular Claude Code frameworks — `get-shit-done`, `oh-my-claudecode`, `ccpm`, or any of the *"N skills + M agents in one repo"* bundles — you have felt the magic. You type something, Claude picks the right skill, things happen. It feels like the tool genuinely understands your workflow.

Pause, and ask the uncomfortable question: **does that skill actually understand *your* codebase, or is it applying generic advice to your code and hoping for the best?**

Consider a concrete scenario. You install a popular `/security-review` skill. It runs a generic checklist: SQL injection, XSS, CORS, secrets in code, auth. Useful. But:

- You wrote a custom middleware that intercepts every request and requires a non-empty `id` on any `DELETE`. The generic skill has never heard of it.
- Your project has a convention that all outbound HTTP must go through a single `httpclient.Do` wrapper with retry and tracing. The generic skill is blind to it.
- You enforce that `internal/` packages cannot be imported from `cmd/`. Again, invisible to a framework that does not know your repo.

A generic skill flags the textbook problems. A skill customized to your codebase flags *your* problems — the ones the review actually needs to catch. Magic is what gets you started. Customization is what turns Claude Code from a 2× tool into a 10× tool. *"N skills × M agents"* is impressive on a GitHub README, but it is only the floor of what is possible when you own the repo.

And the entire "magic" feeling hinges on one mechanic: **how those frameworks trigger their skills and agents.** This is the knob you will need to tune the moment you start customizing.

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

- **Implicit trigger is where "magic frameworks" live.** When a popular Claude Code setup feels like it reads your mind, it is not mind-reading. Someone engineered the skill description, the `when_to_use` hints, the CLAUDE.md phrasing, and the path scope so the model reliably auto-invokes on the right prompts. That is real work — and it is failure-prone. A description has to win the attention competition against every other skill in the budget.
- **Explicit trigger is a safety lever, especially on weaker models.** Auto-trigger assumes the model is smart enough to match a prompt to a description correctly. On smaller, cheaper, or faster models, auto-trigger misfires more often — irrelevant skills invoke, or relevant ones get skipped. Forcing `/command` removes the guessing. Try the same skill set with a Haiku-tier model and an Opus-tier model and you will see the gap.
- **Actions with side effects should almost always be explicit.** If a skill deploys, sends messages, mutates shared state, or spends money, you do not want "Claude decided to" in the post-mortem. `disable-model-invocation: true` is cheap insurance.
- **Pure knowledge injection is fine implicit.** A skill that says *"when touching billing code, here are the invariants"* is ideal auto-trigger material — scope it with `paths:`, hide it from the menu with `user-invocable: false`. Users should not have to memorize `/billing-context`.

---

## Decision tree — "where does this knowledge live?"

Start from the top. The first match wins.

```
Is it derivable from `git log` or by reading the code?
  └── YES → store NOTHING. Don't pollute context.

Is it a MUST / MUST NOT that's always true, regardless of task?
  ├── Machine-checkable (grep, lint)? → rule + hook
  └── Human-enforced only?            → rule

Is it a "when user says X, do Y₁ → Y₂ → Y₃" workflow?
  └── YES → skill

Does answering it require reading >2000 lines of source?
  └── YES → subagent (isolate from main context)

Is it a fact about the user or project that changes over time?
  └── YES → memory

Is it style / tone / philosophy spanning the whole repo?
  └── YES → CLAUDE.md
```

If nothing matches, you probably don't need to persist it.

---

## What's next in this repo

This is the initial scaffold. Coming soon:

- [ ] `docs/anti-patterns.md` — the 6 most common mis-placements
- [ ] `docs/lifecycle.md` — when to promote memory → rule, demote CLAUDE.md → rules/
- [ ] `examples/` — one minimal, sanitized example per primitive
- [ ] Case study — one real flow (test authoring) split across all 5 layers, with reasoning

Issues and PRs welcome. If you disagree with a placement, open an issue — the point is to surface the tradeoffs.

---

## What this repo is NOT

- Not an "awesome-claude" list. Those exist.
- Not a tutorial on installing Claude Code. See [official docs](https://docs.claude.com/en/docs/claude-code).
- Not prompt-engineering tips. Out of scope.
- Not opinions on model choice. Too volatile.

## License

MIT — see [LICENSE](./LICENSE).
