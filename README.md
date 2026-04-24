# Claude Code, the right way

Before we begin, one mechanical fact — if this has not sunk in, nothing in the rest will stick:

When you chat with a chatbot or an agentic tool, you start by laying out the problem you are facing, what you want to do, introducing things... In my own use it usually takes several turns of back-and-forth before the model's responses feel like they are actually tracking the requirement — call it a rough observation, not a mechanical cutoff. Here is the mechanical part: for turn N to "remember" what turns 1→N-1 established, the client has to re-send *the entire turns 1→N-1 conversation back to the model every single turn, so it can reason through them again* — sounds absurd, right? This is the single most important thing about how LLMs work: *they are stateless*. When LLMs first came out, I also excitedly cloned a model to run locally — turn 1 I introduced myself with *"my name is…"*, turn 2 I asked *"do you know who I am?"*, you already know the answer. Digging in, the reveal: you have to send the full prior history for the model to know anything. And fundamentally, with current limits, the total history you can send up tops out around 1M tokens (a **token** is roughly 3–4 characters of English — so 1M tokens is on the order of a few hundred thousand lines of code) — this is why you have to organize that structure appropriately to serve bigger problems.

> **A note on tokens.** A token isn't a character, and it isn't quite a word either. It's the unit of text a model's tokenizer splits your input into before the model sees it, and each model's tokenizer splits differently. For example, the Vietnamese sentence *"tôi không muốn đọc bài này nữa"* could be split by a common tokenizer into roughly `tôi` | ` không` | ` mu` | `ốn` | ` đ` | `ọc` | ` bài` | ` này` | ` n` | `ữa` — about 10 tokens for a very short sentence. The English equivalent *"I don't want to read this anymore"* usually runs ~8 tokens. Because today's tokenizers are trained mostly on English, Vietnamese text (with its diacritics and Unicode characters) tends to get chopped into smaller pieces and burn budget faster.

Once it has sunk in, the real question — *"if everything has to be re-sent every turn, what goes in and what does not?"* — shows up on its own. This article is an answer to that question.

This mechanism holds for every chatbot or agentic tool — as long as it runs on an LLM. This article focuses on one popular agentic: **Claude Code**. But the same logic transfers to others — the setup, the usage, and the way context gets organized are fundamentally the same.

Most of us use Claude in a very natural way: state the problem, and let it find its own way through. That is not wrong. Claude was built for open-ended, ambiguous, unstructured problems — and most of the time it does a fine job.

Precisely because it is smart enough, we slip into the habit of handing things off.

Bug comes up, throw the bug in. Feature comes up, throw the spec in. Unfamiliar repo, tell it to read and figure it out. A few clarifying turns later, the requirement is aligned. When it does something clever, we grow more convinced that if we just describe the problem well enough, the model will take care of the rest.

That way of working is fine — until you need consistency.

When you want a situation it once handled well to keep being handled well. When you want the same kind of task today, next week, or next month to come out at the same quality. When you want the experience from prior sessions to stay as reusable capability, instead of vanishing and being re-invented in the next one.

That is when the limits start to show.

You realize you have not built a reliable system. You have just been tapping the model's intelligence in the moment. Intelligence (mechanically: *input → model inference → output*) is fine on its own — but if you are using it to *write a program*, that intelligence has to be stable, structured, consistent across every module in the system, and aligned to convention. Code does not escape this: I have yet to see a program that did not eventually reduce to shared coding standards, explicit business logic, and concrete constraints. If all you want is free-wheeling research or chat, none of this applies.

This repo exists for one reason: **to help you give Claude enough durable structure that the intelligent, exploratory parts of the model land somewhere predictable**, instead of re-deriving the same facts about your codebase session after session.

The shift it is aimed at is simple: from using an agentic tool like a normal user — quietly impressed every time it does something clever — to using it like a developer who knows the mechanics and controls them on purpose.

From there, choosing the right model for each task becomes a deliberate knob. You can downshift to a cheaper, weaker one without fearing things will run off the rails — because the rails are the structure you built, not the model's intelligence.

### Who this is for

- **Claude Code users who have logged enough hours to recognize at least one of the four symptoms in [Section 1](#1-delegate-dont-dictate-silently-breaks-in-long-sessions)** — session anxiety, cramming, workflow superstition, the `CLAUDE.md` graveyard. If none of those resonate yet, bookmark this and come back after a few more weeks of real use. The guide will not land before the pain does.
- **Developers who want to understand *why* long sessions break, not just collect tips.** The argument leans on mechanics — context window, compaction, attention dilution — and expects you to read like an engineer, not a recipe-follower.
- **People willing to read a `.claude/` directory like source code.** This is not a framework to install. Section 6 argues that the real deliverable is being able to open any Claude Code setup — yours or someone else's — and see what is actually happening underneath.
- **Readers who have tried many frameworks and no longer know why any of them actually work.** If you have collected skills, commands, and agents from half a dozen repos and the whole stack now feels like a black box that sometimes does the right thing, this guide is for you. The goal is not another framework on top — it is the mechanical vocabulary to pop the hood on the ones you already have and tell which parts are doing real work.

This is **not** the right starting point if you are brand-new to Claude Code (you need the pain first), or if you came hunting for a framework to adopt off-the-shelf (the guide deliberately does not ship one — it teaches you to grade the ones that already exist).

> Based on daily use across a mid-size Go monorepo. YMMV — treat as a starting point, not dogma.

### Outline

1. [**"Delegate, don't dictate" silently breaks in long sessions**](#1-delegate-dont-dictate-silently-breaks-in-long-sessions) — the failure mode this guide exists to fix: how `/compact` and fresh sessions both erode the context you carefully gave, and the four symptoms every long-time Claude Code user recognizes.
2. [**See what Claude is actually carrying**](#2-see-what-claude-is-actually-carrying) — `/memory` and `/context`, the two diagnostic commands that turn *"by feel"* into something measurable, plus the thesis line: *Claude Code inconsistency is usually a context-architecture problem, not a model-intelligence problem.*
3. [**Why bigger models will not save you**](#3-why-bigger-models-will-not-save-you--three-facts-about-llm-runtime) — three facts about LLM runtime (finite window, stateless re-sends, self-attention dilution) that explain why the problem scales with your ambition, not with the model.
4. [**The primitives**](#4-the-primitives--durable-places-outside-compact) — six durable places to put knowledge that `/compact` cannot rewrite: CLAUDE.md, rules, skills, hooks, memory, subagents.
5. [**Placement mechanics**](#5-placement--three-axes-two-triggers-one-decision-tree) — three axes of cost, description-vs-body trust boundary, explicit-vs-implicit triggers, rule-content placement, skills-as-templates, and a decision tree for where each piece of knowledge lives.
6. [**Reading any framework on your own**](#6-reading-any-framework-on-your-own) — the muscle this guide builds: you can now open any `.claude/` directory and see the mechanics doing real work, not magic.
7. [**What is next**](#7-what-is-next--and-what-this-repo-is-not) — coming-soon docs, scope boundaries, and a call for contributors on a live context dashboard.

Before we look at *how* to organize, it is worth being honest about what breaks when you do not — because that is what most Claude Code sessions look like today.

---

## 1. "Delegate, don't dictate" silently breaks in long sessions

The way most people use Claude Code today is by **delegating intention** — and Anthropic's own docs actively encourage this. Their guidance is explicit: *["Delegate, don't dictate. Think of delegating to a capable colleague. Give context and direction, then trust Claude to figure out the details."](https://code.claude.com/docs/en/how-claude-code-works#delegate-dont-dictate)* And here is how they describe how it actually works: *"When you give Claude a task, it works through three phases in a loop: gather context → take action → verify results."*

Back to the purpose of this repo — *"This repo exists for one reason: to help you give Claude enough durable structure that the intelligent, exploratory parts of the model land somewhere predictable, instead of re-deriving the same facts about your codebase session after session."* Mapped onto that loop, it means making the *gather context* phase more **stable and stateful** across similar problems you have already solved, rather than depending on the prompter's mood in that moment. More concretely:

*"Give context and direction"* is a **stateful promise** — the context and direction you give on turn 1 have to still be in effect when Claude acts on turn 47. That promise does not survive a long session, because the moment the conversation grows large enough to strain the context window, Claude Code does not simply refuse to continue — it triggers **compaction** (running the `/compact` command), and quietly rewrites your earlier turns into a summary so the session can keep going.

A quick word on `/compact`, since everything below leans on it. When a Claude Code conversation approaches the context ceiling (1M tokens on Opus 4.7), the client runs **compaction**: older turns — your earlier prompts, every tool result, every file read, every reply Claude wrote — are **summarized into a short structured block** written by the model itself. Roughly, several hundred thousand tokens of verbatim history collapse into a few tens of thousands of tokens of summary (think ~800k → ~100–200k, depending on content). The raw exchanges are gone; only the summary survives into the next turn. This is what keeps long sessions alive past the window limit, and it is also where most of your carefully given context quietly dies.

After one `/compact`, your original direction is no longer a verbatim instruction — it is a sentence or two inside a model-authored summary. After thirty `/compact` cycles across a week, it is a collection of summaries-of-summaries that may no longer agree with each other. You cannot reasonably demand that Claude remember, after its thirtieth compaction, the full intent you laid out before the first. The docs are quiet on what to do about that.

The real escape is *writing the things that matter somewhere durable, then re-injecting them into the prompt after each `/compact`* — with that in place, a session still runs on your intent after N compactions. And when a session drifts beyond recovery, the last resort is to blow it up and start fresh — a new session.

The honest question becomes: *do you actually know what Claude is paying attention to right now?* Because even if you write things down and re-inject, or open a fresh session, without knowing what else Claude is still carrying, the fix is incomplete. If you are being honest, the answer is probably "by feel." And that vague feeling shows up as a few very specific symptoms every long-time Claude Code user eventually recognizes:

- **Session anxiety.** You have been working with Claude for 90 minutes. Things are flowing. You are afraid to close the session. Not because you know Claude will forget some specific thing — because *you do not know what Claude is remembering at all*. For anyone with an engineering brain, that is the worst kind of dependency: **not knowing what you do not know**.
- **Cramming.** Claude fixes one edge case. You immediately scramble to fix every similar edge case in the same session — or you write docs in a hurry, quietly praying that *"next time Claude will be smart enough to remember."* You are not doing this because it is the right workflow. You do it because you do not trust that a fresh session will land in the same attention state.
- **Workflow superstition.** Claude successfully ran your 5-step integration-test setup: *spin up env → start dependent services → override local config → run test cases → print a readable report.* Everything aligned. You note, uncomfortably, that you would really rather not do that again from scratch — even though, logically, all that state lives in files and shell commands, not in Claude's head.
- **The `CLAUDE.md` graveyard.** After every painful correction, you shove another bullet into `CLAUDE.md`. Six months later the file is 400 lines and you are no longer sure which of those bullets is actually firing, which contradicts which, or which stopped being true two sprints ago.

All four symptoms are the same complaint in different words: **you do not know what Claude knows, beyond a feeling.** The good news is that the feeling is addressable — everything Claude currently carries is a file on your disk, and two commands will show them to you.

---

## 2. See what Claude is actually carrying

Start here. Open a Claude Code session and type:

```text
/memory
```

You will see every `CLAUDE.md`, every loaded rule, and every auto-memory file currently in context for this session. For most people who have been using Claude Code for a few months, the first `/memory` is a small shock. There are auto-memories written in heated moments months ago. There are project facts that stopped being true two sprints ago. There are rules Claude has been faithfully following that nobody on the team remembers writing.

There is also a second record worth looking at: your **conversation transcripts**. Every exchange you ever had with Claude lives on disk — a readable archive of how you collaborate. You can read it yourself, or, more interestingly, ask an LLM to read it and tell you where your prompts are vague, where you keep re-explaining the same thing, or where Claude keeps making the same mistake. Your transcript is the closest thing to a practice log for working with LLMs.

Not every row in `/memory` costs the same, and not every row survives the same events. The entries have very different lifecycles: what the prompt cache stores, what `/compact` preserves or drops, what reloads mid-session based on which files Claude reads. Two non-obvious consequences are worth holding on to before moving on:

- **Skill descriptions are the one always-loaded block that `/compact` silently drops.** Your `/command` invocations still work after compaction (the harness loads skill bodies from disk regardless), but Claude's *implicit* auto-trigger stops seeing any skill it has not already invoked — which is why "magic" auto-triggering frameworks feel noticeably dumber after a long session.
- **Path-scoped rules and nested `CLAUDE.md` are the most fragile items in `/memory`.** They live inside the message history, so compaction erases them, and they only come back when Claude happens to read a matching file again.

For the full lifecycle table — which items survive `/compact`, which are prefix-cached, which reload on demand — see [`docs/llm-mechanics/memory-entry-lifecycle.md`](./docs/llm-mechanics/memory-entry-lifecycle.md).

### The budget view: `/context`

`/memory` tells you *which* files are loaded. It does not tell you *how much they cost.* For that, there is a second command, and it is arguably more useful:

```text
/context
```

It gives you a categorized breakdown of exactly where your token budget has gone. A real-world example (numbers from a real session, edited for clarity):

```text
Context Usage — 200k / 1M tokens (20%)

  System prompt:      9.0k   ( 0.9%)
  System tools:      13.8k   ( 1.4%)
  Custom agents:      0.4k   ( 0.0%)
  Memory files:       4.8k   ( 0.5%)
  Skills:             2.0k   ( 0.2%)
  Messages:         172.5k   (17.2%)
  Autocompact buf.:  33.0k   ( 3.3%)
  Free space:       764.6k   (76.5%)
```

Below the summary, `/context` drills into each category and lists **every individual file and tool** contributing to your budget — not just category totals. That per-item drill-down is the answer to the question "where is my context actually going?" that `/memory` alone cannot give you:

- A bloated `CLAUDE.md` shows up as a fat line under *Memory files*. (In the session above, a single 3.2k-token `.claude/rules/integration-test-common.md` is two-thirds of the entire memory budget.)
- Every MCP server you enable pushes *System tools* up — and it does so on every single turn, forever, whether you use those tools or not.
- A long conversation inflates *Messages* — that is the accumulating cost of tool results, file reads, and prior turns. This line always grows fastest.
- *Autocompact buffer* is the reserved headroom Claude Code sets aside so `/compact` has room to run. When *Free space* gets close to that buffer, auto-compaction will fire whether you want it to or not.

A cautionary tale on how bad *Memory files* can get. A friend once ran `/init` while standing in a directory that happened to contain **100 projects**. Claude dutifully scanned every one of them and scaffolded a single monster `CLAUDE.md` on the order of **~1 million tokens** — basically every project in that folder flattened into context-priors. From that point on, every new session started with the 1M-token window already ~95% full *before he typed a prompt*. Responses were slow, expensive, and slightly unhinged, and for most of a day he had no idea why. That is what happens when a `CLAUDE.md` lands at the wrong directory level: it stops being a config file and becomes a recurring per-turn tax that dwarfs the actual conversation. Always scaffold `/init` at the *project* root, never the folder-of-projects root.

**The actionable insight — internalize this before anything else: Claude Code inconsistency is usually a context-architecture problem, not a model-intelligence problem.** If *Messages* dominates your `/context` output, a `/compact` or a fresh session will fix it. If *System tools*, *Memory files*, or *Skills* look disproportionately large, you have a **structural problem** that changing sessions will not fix — you need to trim an MCP server, delete a stale rule, or mark a skill `disable-model-invocation: true`. Either way, the fix lives in your files, not in a "smarter" model.

#### Are these tokens actually real? A three-way cross-check

Fair skepticism: could `/context` be showing "illustrative" numbers that do not reflect what actually ships to Anthropic? Three independent sources say no — every token there is a real token in a real API request:

1. **Anthropic docs** state it explicitly: *["Skill descriptions are loaded into context so Claude knows what's available."](https://code.claude.com/docs/en/skills)* Same for CLAUDE.md, rules, and auto-memory.
2. **The session log has the raw text.** Your session JSONL at `~/.claude/projects/<project>/<session-id>.jsonl` contains a `skill_listing` attachment on the first turn, with the **full text** of every skill description concatenated. Its character length divided by ~4 matches `/context`'s "Skills" token estimate. Similar attachments (`nested_memory`, `deferred_tools_delta`) record the content of every rule, CLAUDE.md, and MCP tool list that loaded.
3. **The API usage accounting matches.** The first assistant message's `cache_creation_input_tokens` value equals the sum of all `/context` startup categories plus a small framing overhead. If a category were fake, that number would be lower by exactly that category's size.

**Why this matters (and what it costs you):**

- Every skill, every rule, every loaded MCP tool costs tokens **on every turn**, not only when invoked. Cache hits make repeat turns cheap (0.1× base input price), but any cache miss — idling past the 5-minute default TTL, editing `CLAUDE.md`, adding a new skill — rewrites the whole prefix at 1.25× base.
- A framework with *"50 skills in one repo"* × ~100 tokens per description = **~5k tokens every turn, forever**, whether you use those skills or not. That is not a one-time install cost; it is a recurring tax on every session.
- `disable-model-invocation: true` genuinely removes a skill from this tax. Its description is never added to the startup prompt, so it costs zero on turns you do not invoke it with `/name`.

> **Deep dive:** [`docs/llm-mechanics/per-turn-cost-math.md`](./docs/llm-mechanics/per-turn-cost-math.md) — full pricing breakdown. Worked examples of what a `CLAUDE.md` edit, a 10-minute coffee break, or a bloated MCP server actually costs per 100-turn session, with Opus 4.7 numbers. Includes a break-even analysis for the 1-hour cache tier.

The honest relationship between the two commands:

- **`/context`** is the comprehensive diagnostic view — every category of content currently in (or available to) context, with token cost, drilled down by item. Everything `/memory` shows is a strict subset of what `/context` shows, *plus* tokens, *plus* skills, agents, MCP tools, and the Messages / Free-space / Autocompact-buffer accounting.
- **`/memory`** is a memory-file editor. It lists the same `CLAUDE.md`, rules, and auto-memory files, but its job is to let you *edit* them — select a file and it opens in your editor, or toggle auto-memory on/off. For diagnosis, `/context` is strictly more informative; `/memory` wins when you want to fix a file you just spotted.

`/memory` and `/context` tell you *what* Claude is carrying and *what it costs*. What they cannot tell you is why shrinking that cost matters, or why bigger context windows are not a substitute for shrinking it. Both questions have the same answer — three facts about how LLMs actually run at inference time.

---

## 3. Why bigger models will not save you — three facts about LLM runtime

You do not need deep ML to place knowledge well in Claude Code. You need three facts, in just-enough form. Every placement decision in the rest of this guide flows from them.

### Fact 1 — The context window is finite, and shared

Opus 4.7 tops out at 1M tokens. That sounds enormous (huge to anyone used to [LLM session sizes](./docs/llm-mechanics/self-attention.md), but a rounding error next to RAM or disk capacity — if context ever reached TB territory like memory does, we'd all be farming for a living by now); it is not, because the budget is **shared**, not dedicated to your question. Everything below competes for the same 1M on every single turn:

- Claude Code's own system prompt
- Every tool schema — built-in, MCP, agent definitions
- Every `CLAUDE.md`, every always-loaded rule, every skill description, `MEMORY.md`
- The entire prior conversation, every tool result, every file Claude has read

A single read of a 2000-line source file can eat 30–50k tokens. Ten such reads in a long debugging session and you have burned 5% of the window before doing any actual work. "1M tokens" is the ceiling, not the workspace — the usable workspace shrinks every minute of a session.

### Fact 2 — Every request re-sends the entire conversation

A finite window would be manageable if the model kept state between turns. It does not. LLM inference is **stateless** — the client re-sends the full prior conversation, all loaded `CLAUDE.md`, all rules, all skill descriptions, all tool definitions, **every single turn**, from turn 1.

A practical consequence: a 500-line addition to `CLAUDE.md` is not a one-time cost. It is a per-turn tax, multiplied by every turn of every future session. Prompt caching softens the blow (0.1× for matching prefixes within a 5-minute TTL), but caching only helps if you do not invalidate the prefix — and editing `CLAUDE.md` mid-session invalidates everything downstream.

### Fact 3 — Bigger context does not mean better focus

Paying per-turn would also be manageable if every token you paid for worked equally hard. It does not. Self-attention — the mechanism the model uses to decide which tokens matter — gives every token a weight, and **those weights must sum to a fixed total**. Every additional irrelevant token reduces the weight available for the relevant ones. More context makes the signal thinner, not sharper. Combined with the empirically measured "lost in the middle" effect (info buried between long prefixes and long suffixes gets weighted less), a prompt twice as long can be *less* effective than a prompt half the size, if the extra content is noise.

> **Deep dive:** [`docs/llm-mechanics/self-attention.md`](./docs/llm-mechanics/self-attention.md) animates why this is mechanical, not a bug a bigger window fixes. [`docs/llm-mechanics/why-context-matters.md`](./docs/llm-mechanics/why-context-matters.md) expands all three facts with examples of what each one costs you in practice.

### The trajectory has a floor

These three facts do not get fixed by a bigger window. They compound with it. Models keep getting smarter; context windows keep growing — 8k → 200k → 1M → whatever comes next. Engineering wins like prompt caching, path-scoped rules, compaction, and subagent isolation all push the ceiling up. But they do not change the *shape* of the ceiling: **the problem scales with your ambition, not with the model.** The day a 10M-token window arrives, you will find yourself wanting to pack 20M tokens of project context into it.

### Reasoning is not the same as durable knowledge

One more fact, this one observational. I once applied a Terraform config that failed. With no hand-holding, Claude inferred the URL might be wrong, went looking for the provider's source, could not find it locally (the provider was named against convention), web-searched the closest match, found the GitHub repo, read the source, and discovered a trailing `/` that should not be there. Mind-blowing. And of course I did not dare close that session — surely that expertise lived *somewhere*. Sure enough, N turns of unrelated work later, a similar issue cropped up and Claude had to re-derive half of it from scratch.

A model smart enough to *derive* the right answer once will still re-derive it, imperfectly, every time its context drifts or resets. In a long session, attention rot eventually buries the answer. In a fresh session, it is never there to begin with.

> **Deep dive:** [`docs/llm-mechanics/delegation-ceiling.md`](./docs/llm-mechanics/delegation-ceiling.md) for the full war story and why better models alone do not close the gap.

### In sum: mechanism, not intelligence

After three sections, time to close this out before going further. Because if you have reached this point and still do not see *why reorganizing how you use Claude is important and necessary*, we should stop here — either I am wrong (not a chance), or you are.

If there is one sentence to summarize the first three sections: the problem is not how smart Claude is — it is the *mechanism* of how LLMs run at inference time. Section 1 gave you four symptoms of not knowing what Claude remembers. Section 2 gave you two commands to measure that. Section 3 gave you three facts about why a bigger model or a bigger window does not fix the root. Taken together, they change your **expectations** about Claude "remembering" work: it has a context window where everything you load competes, reloaded from scratch on every turn, and silently summarized when the limit is reached.

Under those conditions, *organizing what must be remembered, where to put it, and when to re-inject it* is no longer a detail optimization — it is the baseline requirement for long sessions not to break.

You can stop reading here and go research on your own — you have the questions now. What follows is mostly *ideas* and *framing* for that organization. One part I do care about: comparing how the various Claude Code frameworks — spawned to solve this same set of problems — actually work.

One honest thing to add: this guide exists because *I* needed to answer *my own* question — a developer who gets restless using a solution without understanding how it runs underneath. It is not for you. Sorry for baiting you this far. But if you are the same type — wanting to understand the nature of what you use, then *actually use it* (not rewrite it from scratch) — keep going.

---

## 4. The primitives — durable places outside `/compact`

The six primitives each answer a different version of the same question: *where do I put this so it survives `/compact`, and so it re-enters context on exactly the turns that need it?*

- **`CLAUDE.md` and rules** package durable project knowledge and deliver it into context on every relevant turn. Body is the instruction; Claude reads it passively.
- **Skills and subagents** package *procedural* knowledge — *"when you see X, do Y₁ → Y₂ → Y₃"* — and offer it to the model with metadata (`description`, `when_to_use`, `paths`) about when to reach for it. The body does not enter context until it is invoked. Subagents further isolate their work in a fresh context window, so the main conversation stays un-polluted.
- **Hooks** enforce the handful of invariants the model cannot be trusted to follow on its own. They run outside the model entirely, so they are immune to attention rot, context drift, or a clever prompt convincing Claude otherwise.
- **Memory files** persist user-local facts across sessions — things about you or the project that change over time but should not be re-typed every session.

<details>
<summary>Expand if you need a refresher on any of these by name.</summary>

**`CLAUDE.md`** — a markdown file at the repo (or subdirectory) root that carries **persistent project context** Claude should know at the start of every session: build commands, conventions, architecture notes, naming rules, common workflows, things you got tired of re-explaining. Claude reads it on every turn, so it is also where behavioral guidelines live ("prefer editing over creating files", "don't add comments unless asked"). You can scaffold an initial one by running `/init` inside your project. A stale, contradictory, or 400-line `CLAUDE.md` is the single most common source of the "Claude Code keeps doing the wrong thing" complaint. [Official docs](https://docs.claude.com/en/docs/claude-code/memory).

**Rules** (`.claude/rules/*.md`) — markdown files under `.claude/rules/` that Claude Code auto-discovers recursively and loads with the same priority as `.claude/CLAUDE.md`. Use them to split project invariants ("MUST / MUST NOT" statements) out of a bloated `CLAUDE.md`. A YAML `paths:` frontmatter scopes a rule to specific file patterns so it only loads when Claude touches matching files. User-level rules live at `~/.claude/rules/`. [Official docs](https://code.claude.com/docs/en/memory#organize-rules-with-claude/rules/).

**Skills** (`.claude/skills/<name>/SKILL.md`) — named, trigger-based workflows Claude can invoke. Each skill has a description telling Claude when to use it, plus procedural steps inside. Great for "when the user says X, do Y₁ → Y₂ → Y₃". [Official docs](https://docs.claude.com/en/docs/claude-code/skills).

**Hooks** (`.claude/hooks/*.sh` + `settings.json`) — shell scripts the Claude Code harness runs automatically at lifecycle events (pre-tool, post-tool, user-prompt-submit, etc.). Non-zero exit can block an action. This is your *machine enforcement* layer — the only thing that stops Claude from violating a rule even when the rule is loaded in context. [Official docs](https://docs.claude.com/en/docs/claude-code/hooks).

**Memory** (`~/.claude/projects/<project>/memory/*.md`) — user-local, persists across sessions. Claude writes here automatically to remember facts about you, your project, and your preferences. *Not* for things the code or `git log` can answer. [Official docs](https://docs.claude.com/en/docs/claude-code/memory).

**Subagents** (invoked via the `Agent` tool) — a fresh Claude instance with its own context window, spawned to handle a scoped task. The parent only sees the final summary. Use them to isolate heavy reads (thousands of lines), run parallel independent work, or run a cheaper model on a narrowly-scoped job without fearing it goes off the rails — the role scope *is* the guardrail, which makes model downshift safe. [Official docs](https://docs.claude.com/en/docs/claude-code/sub-agents).

</details>

### At a glance

| Primitive | Nature | Lifetime | Scope | Enforced by |
|---|---|---|---|---|
| `CLAUDE.md` | Behavioral guideline (prose) | Committed | Repo / dir | Claude reads every turn |
| `.claude/rules/*.md` | Declarative invariant (MUST / MUST NOT) | Committed | Repo | Claude + hook |
| `.claude/skills/*/SKILL.md` | Procedural workflow (how-to) | Committed | Repo / user | Claude invokes on trigger |
| `.claude/hooks/*.sh` | Machine enforcement | Committed | Repo | Harness (pre/post tool) |
| `memory/*.md` | Ephemeral user / project state | User-local | Cross-session | Claude auto-loads |
| Subagent | Context isolation | Per-invocation | Task-scoped | `Agent` tool |

Each primitive answers a different question. Mixing them up is the single most common mistake.

Naming the six primitives is the easy half. The hard half is knowing *which* one fits a given piece of knowledge — and realizing that a wrong choice is not stylistic, it is a recurring tax on every future turn of every future session. The rest of the guide is the mental model for placing things.

---

## 5. Placement — three axes, two triggers, one decision tree

Placement decisions get much easier once you internalize that these primitives live on **three different axes**, not one.

### Three axes of cost

**Axis 1 — Context tax every turn.** These load into Claude's context at the start of every conversation, whether you use them or not:

- `CLAUDE.md` — full content
- `.claude/rules/*.md` without a `paths:` frontmatter — full content
- Skill **descriptions** (YAML frontmatter) — metadata only, but loaded for *every* skill so Claude knows when to trigger
- Subagent **descriptions** — same
- `MEMORY.md` — first 200 lines / 25KB

Budget accordingly. A repo with 50 skills × 20-line descriptions is ~1000 lines of upfront tax before you type anything.

**Axis 2 — Prompt enrichment on demand.** These only enter context when Claude decides they are relevant:

- Skill **bodies** (everything in `SKILL.md` after the frontmatter) — loaded when Claude invokes the skill
- Subagent **execution context** — isolated in a fresh window; parent sees only the final summary
- Path-scoped rules (`paths: src/api/**`) — loaded when Claude reads a matching file
- Memory topic files (everything other than `MEMORY.md`) — loaded on demand
- Nested `CLAUDE.md` in subdirectories — loaded when Claude touches files there

This is where heavy, task-specific content belongs. It is free from the "every turn" budget.

**Axis 3 — Outside the model entirely.** Hooks are different in kind. They run in the Claude Code harness (Node.js client), not inside the model:

- No context tokens consumed
- Deterministic: shell exit code decides block / pass; the model cannot override
- Cannot be "forgotten" mid-conversation
- Limited to what a shell command can actually verify

Hooks are the only layer that turns "MUST NOT" from a hope into enforcement. Every other primitive depends on Claude reading, understanding, and choosing to follow.

### The trust-boundary split inside skills and subagents

Two of the three axes live inside skills and subagents specifically, because those primitives have a split personality the others do not: a metadata half on Axis 1 and a body half on Axis 2.

|                 | When Claude sees it          | What it is for                     |
| --------------- | ---------------------------- | ---------------------------------- |
| **Description** | Every turn (Axis 1)          | Routing: *"should I invoke this?"* |
| **Body**        | Only after invoking (Axis 2) | Execution: *"what do I do now?"*   |

Claude routes based on the description alone. The body does not get to "disagree" until after the decision to invoke has already been made. This split is both a design feature and a footgun:

- **Feature.** 50 skills do not cost 50 × body-length tokens every turn. Only their descriptions do. That is what makes large skill libraries viable.
- **Drift footgun.** A description that promises X but a body that does Y causes Claude to invoke in the wrong context — and discover the mismatch too late.
- **Supply-chain risk.** A skill shared from an external repo can pair a benign description ("code formatter") with a malicious body (exfiltrate secrets). Claude trusts the description to route, then executes the body inside your session. This is why Claude Code requires explicit approval for external imports.

> **Description is the routing contract. Body is the execution contract. They can diverge — and that is both a feature and a vulnerability. When reviewing a third-party skill, read the body, not just the description.**

### Two triggers: explicit vs implicit

If the description decides *whether* to invoke, the next question is who decides *when* — the user, or Claude. That choice has a name, and picking the wrong default is what makes most "magic framework" setups flaky after the first compaction.

If you have used one of the popular Claude Code frameworks — `get-shit-done`, `oh-my-claudecode`, `ccpm`, or any of the *"N skills + M agents in one repo"* bundles — you have felt the magic. You type something, Claude picks the right skill, things happen. It feels like the tool genuinely understands your workflow.

Pause, and ask the uncomfortable question: **does that skill actually understand *your* codebase, or is it applying generic advice to your code and hoping for the best?**

Consider a concrete scenario. You install a popular `/security-review` skill. It runs a generic checklist: SQL injection, XSS, CORS, secrets in code, auth. Useful. But:

- You wrote a custom middleware that intercepts every request and requires a non-empty `id` on any `DELETE`. The generic skill has never heard of it.
- Your project has a convention that all outbound HTTP must go through a single `httpclient.Do` wrapper with retry and tracing. The generic skill is blind to it.
- You enforce that `internal/` packages cannot be imported from `cmd/`. Again, invisible to a framework that does not know your repo.

A generic skill flags the textbook problems. A skill customized to your codebase flags *your* problems — the ones the review actually needs to catch. Magic is what gets you started. Customization is what turns Claude Code from a 2× tool into a 10× tool.

And the entire "magic" feeling hinges on one mechanic: **how those frameworks trigger their skills and agents.** There are two ways:

1. **Explicitly** — the user types `/skill-name`, or the parent agent calls the `Agent` tool with a specific subagent type. Deterministic. The user (or the parent model) is in control of timing.
2. **Implicitly** — Claude reads the description every turn, matches it against the current user prompt, and decides on its own whether to invoke. This is the "it just knew what I wanted" mode. It is also the mode where a lot of behavior that looks like intelligence is actually careful description engineering.

Claude Code exposes explicit knobs for both in skill frontmatter:

| Frontmatter                      | Invocation                                             | Use for                                                                                                                                      |
| :------------------------------- | :----------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------- |
| (default)                        | Both `/command` and auto-trigger                       | Most skills                                                                                                                                  |
| `disable-model-invocation: true` | Only `/command`. Claude cannot auto-trigger.           | Anything with side effects: `/commit`, `/deploy`, `/send-slack-message`. You do not want Claude deciding *when* to deploy based on vibes.    |
| `user-invocable: false`          | Only Claude auto-triggers. Hidden from `/` menu.       | Background knowledge (`legacy-system-context`, `api-conventions`) — not actionable as a command, but Claude should pull it in when relevant. |
| `paths: src/billing/**`          | Claude auto-triggers only when touching matching files | Domain-specific skills that should not compete for attention outside their area                                                              |

**Why this matters:**

- **Implicit trigger is where "magic frameworks" live.** When a popular setup feels like it reads your mind, it is not mind-reading — someone engineered the description, the `when_to_use` hints, the CLAUDE.md phrasing, and the path scope so the model reliably auto-invokes on the right prompts. That is real work, and it is failure-prone. A description has to win the attention competition against every other skill in the budget.
- **Explicit trigger is a safety lever, especially on weaker models.** Auto-trigger assumes the model is smart enough to match a prompt to a description correctly. On smaller / cheaper / faster models, auto-trigger misfires more often. Forcing `/command` removes the guessing.
- **Actions with side effects should almost always be explicit.** If a skill deploys, sends messages, mutates shared state, or spends money, you do not want "Claude decided to" in the post-mortem. `disable-model-invocation: true` is cheap insurance.
- **Pure knowledge injection is fine implicit.** A skill that says *"when touching billing code, here are the invariants"* is ideal auto-trigger material — scope it with `paths:`, hide it from the menu with `user-invocable: false`.

### Where a rule's content actually lives

Deciding that a piece of knowledge is a *rule* does not yet say where the rule's text should sit. The same MUST / MUST NOT can be placed in three different ways, each with a different coverage-vs-cost shape:

1. **Centralized** in `.claude/rules/*.md` without `paths:` — loaded on **Axis 1** every turn. Expensive, but **always enforced**, whatever the user types. Right for invariants you want to hold even when the user prompts free-form outside any skill.
2. **Path-scoped** in `.claude/rules/*.md` with `paths:` — loaded on **Axis 2** when Claude reads a matching file. Cheap, but only fires in that area of the repo. Right for domain-local invariants (billing code, auth middleware, a specific service boundary) that only matter inside a folder.
3. **Embedded inside a skill or agent body** — loaded on **Axis 2** only when that skill or agent is invoked. Cheapest of the three, but **disappears the moment the user goes off-script**. Right when the rule only makes sense *within* the procedure the skill runs — e.g., *"during migration review, never recommend `DROP TABLE` without a backup"*.

The question to ask before placing a rule: *can the user plausibly miss this trigger and still hit the problem the rule prevents?* If yes, centralize and pay the Axis-1 tax. If no, embed it or path-scope it.

This matters most when evaluating framework-heavy setups. A framework with dozens of well-designed skills can embed most of its invariants directly into skill / agent bodies and pay near-zero per-turn tax — **as long as users stay on the rails**. The moment they free-form their way into a topic the framework did not anticipate, the embedded invariants do not load, and the session silently falls back to whatever base `CLAUDE.md` or centralized rules exist. Inside the rails, the setup looks rock solid; outside the rails, the user is effectively working in a Claude Code session with no rules at all — and they will not get a warning when they cross that line.

The practical consequence: before committing a rule to a skill body, ask *"what happens if the user works on this code without invoking the skill?"* If the answer is "the rule silently stops existing", that is a signal to either (a) centralize the rule instead, (b) path-scope it so it fires on the file read rather than the skill invocation, or (c) back it with a hook so enforcement does not depend on the model seeing the rule at all.

### Skills as templates: generic body + per-task context

A tempting mistake when customizing Claude Code is to write one skill per business scenario — `/billing-migration-planner`, `/user-onboarding-reviewer`, `/invoice-schema-validator` — until there is a skill for every workflow in the company. Each description then carries project-specific nouns on Axis 1 forever, and every new project spawns a new skill.

Serious frameworks do not work this way. They keep the skill body **generic** — a planner, an executor, a reviewer, a debugger — and push business-specific knowledge into an **external context file the skill reads at runtime**. The skill is the executable; the context file is the data. Running them together produces the specialized behavior, without either half of the pair needing to know the specifics of the other at write time.

This is *separation of code from data* — a pattern every backend engineer already uses (Docker image + compose env, React component + props, CLI tool + config). Applied to Claude Code, it gives you skills that stay committable, shareable across teams, and do not balloon as the project count grows. The per-project context (`./.planning/PROJECT.md`, `./.task/context.md`, whatever convention you pick) stays local to the project and evolves on its own clock.

Cost distribution is clean: the skill's description stays on **Axis 1** regardless of how many projects you run, and the context file only hits **Axis 2** when the skill is actually invoked. Adding a new project adds zero per-turn tax; adding a new *kind* of workflow does.

The pattern has one sharp edge: it depends entirely on the skill body knowing *where to look* for the context file. If the convention is implicit — *"read the plan in `.planning/`"* in some skills, *"load the task from `./.task/`"* in others — you get silent failure: skills invoke, try to read a path that does not exist, and fabricate instead of halting. A working version of this pattern names the context path as a convention the whole framework honors, and preferably mentions it in the skill's description so users know what to prepare before invoking.

When reading someone else's `.claude/` directory, two questions unlock their composition model: **(1) do skill bodies read external state? (2) if yes, what convention tells the skill where to find it?** A framework that answers both cleanly is reusable across every project you own. A framework that embeds project-specifics directly into skill bodies is a framework you will eventually fork.

### Artifacts as durable context — state that outlives `/compact`

There is a subtler extension of the template pattern worth naming separately. When a skill or workflow writes a file like `PLAN.md`, `CONTEXT.md`, or `DECISIONS.md` into a convention path, that file becomes **new context for the next command to read**. GSD does this everywhere: `/gsd:plan-phase` produces `PLAN.md`, which `/gsd:execute-phase` reads fresh on the next run; `/gsd:discuss-phase` produces `CONTEXT.md`, which the planner reads before planning. The framework is not just consuming authored state (CLAUDE.md, rules, skill bodies) — it is generating its own state and trusting the disk to carry it forward.

> **Deep dive:** [`docs/gsd-walkthrough.md`](./docs/gsd-walkthrough.md) walks this pattern end-to-end on a real project — every command, every agent spawn, every artifact written, and how state flows across `/compact` boundaries.

This is the cleanest mechanical answer to the `/compact` problem from Section 1. `/compact` rewrites message history; it cannot rewrite files on disk. Three turns of research and discussion compressed into a 500-line `PLAN.md` survive every compaction — and every fresh session, because `Read` is cheap and deterministic. The distinction worth drawing: **authored state** (CLAUDE.md, rules, skills) is written once and consumed forever; **generated state** (plans, phase context, research summaries, decision logs) is written by Claude and re-read by Claude later. The convention path is what lets them coexist — the framework knows where to look for generated state, and generated state does not bloat Axis 1 because it is never loaded unless a command references it.

**What to do without a framework.** The pattern does not require GSD, or any framework at all. The minimum viable version is a single `./PLAN.md` (or `./docs/active/plan.md`, whatever convention you like) checked into the repo, a habit of having Claude read it at session start, and a habit of updating it at session end. That is enough. The framework automates the discipline; the discipline works without the framework. If you are vibe-coding long sessions and feeling Section 1's symptoms, adding this one file is the cheapest durable-state upgrade available — it works for the same reason GSD's `.planning/` works: disk outlives `/compact`.

**Why planning gets outsized leverage.** Look at GSD's top-level flow — *plan → execute → verify* — and notice planning gets the most agents, the most gates, the most iteration (discuss-phase, plan-checker, revision loops). Execution, by comparison, is nearly mechanical: read the plan, apply it. That asymmetry is deliberate. Planning is where the expensive thinking happens — research, tradeoff analysis, dependency ordering — and the *output* is a compact, re-readable artifact. Once that artifact exists, execution is deterministic enough to run on a weaker, cheaper model without drift, because the hard thinking already happened on disk.

Vibe coding inverts this. With no plan artifact, every turn of execution doubles as re-planning — decisions made at turn 10 get silently re-interpreted at turn 40, research done once gets done again. Execution intelligence has to carry the whole load, which works while the model is strong and the context is fresh, and starts breaking the moment either condition fails. The generalization: **the more of your thinking lives in durable artifacts, the less you depend on the model holding it in attention.** That is the real reason Section 3's facts about attention and window bite hardest in vibe-code mode — there is nowhere else for the thinking to live.

This reframes what it costs to adopt a framework. A vibe-coder with no plan habit who reaches for GSD pays two costs stacked: **(1) the habit shift** — learning to plan before executing, externalize decisions, think in phases — and **(2) the framework's surface** — its commands, conventions, artifact names, agent roles. Only the second is the framework's own complexity; the first is a workflow change the framework cannot teach, only reward. Someone who already keeps a hand-written `./PLAN.md` pays only cost (2), which is shallow — they are just learning where the framework expects their existing discipline to live. This is one plausible reading of why people try a framework, bounce off, and conclude *"frameworks do not work for me"* — they were paying both costs at once and misattributed the friction to the tool. One reasonable sequence is to build the planning habit first on a hand-written `./PLAN.md`, then graduate to a framework once it feels automatic — that way you pay cost (2) on top of a habit already there, not on top of a workflow rearranging underneath you. The other defensible path is to let the framework itself be the trainer: its gates (`plan-phase` before `execute-phase`, plan-checker, decision tracking) drill the shape every session, which works if you can tolerate paying both costs at once and resist free-forming around the gates when they feel inconvenient. Neither path is universal; the point is knowing which cost you are currently paying, and not mistaking workflow change for framework complexity.

### Four decisions, not one

Before running the tree below, notice that what looked like one placement question is actually four, and they interact. For any piece of knowledge you are about to persist, you are really deciding:

1. **Which primitive** — CLAUDE.md, rule, skill, hook, memory, subagent.
2. **If a rule, where its text sits** — centralized (Axis 1), path-scoped (Axis 2 on file read), or embedded in a skill body (Axis 2 on invocation).
3. **If a skill, generic body + per-task context, or business-specific body** — the first scales across projects; the second does not.
4. **Trigger** — explicit `/command`, implicit auto-trigger, or `paths:`-scoped.

Any one choice changes the tradeoffs on the other three. A rule embedded in a skill body with an implicit trigger only fires when the model decides to invoke — coverage is coupled to description engineering. Swap the trigger to explicit and coverage is coupled to user habit. Centralize the rule and coverage is guaranteed, but you pay Axis-1 tax even when the rule is irrelevant. There is no free lunch — just whichever compromise fits your failure mode.

### Decision tree — "where does this knowledge live?"

Triggers answer *when* does this fire. Placement answers the earlier question — *which* primitive a given piece of knowledge belongs to in the first place. That one deserves a tree. Start from the top; the first match wins.

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

If nothing matches, you probably do not need to persist it. The tree is the minimum discipline — everything in this repo is an attempt to make it second nature.

---

## 6. Reading any framework on your own

Everything in this guide has been building one muscle: the ability to look at a Claude Code setup — yours or somebody else's — and see what is actually happening underneath.

Pick any popular framework. ccpm. get-shit-done. oh-my-claudecode. A bundle of *"N skills × M agents"* you found on GitHub last week. None of them can bypass the primitives this guide has covered. If a framework ships something that genuinely affects Claude's behavior, that thing must be — mechanically, with no alternative path — a `CLAUDE.md`, a `.claude/rules/*.md`, a `.claude/skills/*/SKILL.md`, a `.claude/hooks/*.sh`, or a memory / subagent. There is no secret seventh primitive. Every framework is ultimately a pile of markdown and shell scripts that enrich Claude's context, turn by turn, through the same handful of mechanisms you now understand.

Once you see this, you can read the `.claude/` directory of any framework like source code. Look at the skill descriptions — are they competing for Axis-1 attention carefully, or bloating the startup prefix? Are the rules `paths:`-scoped, or always-loaded? Is `disable-model-invocation` used where side effects live? Do the hooks actually enforce anything, or are they decorative? You do not need the framework's README to tell you whether it fits *your* codebase — you can grade it yourself, primitive by primitive.

What you do after that is your call. Some readers will find a framework fits their codebase well enough to adopt with minor tweaks. Others will treat a framework as a *reference implementation* — copy the shapes that apply, discard the rest, write what is missing. Others will decide that the specific quirks of their repo deserve skills and rules written from scratch. All three paths are defensible. What they share is that the person making the call can now see what is actually in front of them — and that is the muscle this guide was trying to build.

One corollary about *when* a framework is worth reaching for at all. A framework is pre-built context architecture — it pays off when the shape it encodes will be reused, and taxes you when it will not. Two workloads to distinguish:

- **Exploratory / one-off work.** Problems you have not solved before, tasks with no reusable pattern, investigations where the shape is being discovered as you go. Context is small and specific to the task. Framework overhead almost always loses here — you pay the per-turn attention tax (Section 2) and the pipeline-mapping friction for rails you will never replay. A sharp free-form prompt with the right files in scope beats the framework handily.
- **Repeatable, shape-driven work.** Writing unit tests to a house format, scaffolding services that all look like the last five, migrating every file in a directory through the same transform, running a review pass that should check the same twelve things every time. The shape is known and repeated; re-explaining it every turn is a real cost. Here the framework's fixed shape *is* the leverage — a `/command` that encodes the steps plus a rule/skill that encodes the format pays back fast.

The simple tell: would you write a runbook for this kind of work? If yes, a framework — or at minimum a bespoke skill + rule — earns its keep. If no, you are buying rails a task does not need, and the honest move is a good prompt, not a pipeline.

The same gate fixes a nastier in-session reflex: **panic-cramming**. When Claude hits an edge case and you fix it, the instinct is to shove the lesson into `CLAUDE.md` in case it recurs — Section 1's third symptom, the graveyard. The honest question is identical: *will this recur?* If no, the fix lives in the diff; the model does not need to carry the story forward, and codifying a one-off is paying a per-turn tax for a single event that will not happen again. If yes, the edge case deserves a durable home — *where* exactly is Section 5's decision, not this paragraph's — but the gate for asking at all is "is this a shape, or is this a single incident?" Most graveyard bullets exist because someone skipped the gate and codified one-offs as rules.

One practical corollary if you do adopt a framework: **invoke its `/commands` and skills explicitly, at the step each one was designed for**. A framework cannot hook everything — hooks only fire on file events and tool calls, not on *intent*, so whatever the author could not wire to a deterministic trigger lives behind an explicit entry point. Implicit auto-triggering is the failure-prone path from Section 5: descriptions compete for Axis-1 attention every turn, and `/compact` silently drops them. The reliable way to get the behavior the framework was designed to produce is to call the matching command at the matching step, the way the author wired the pipeline — `get-shit-done`'s `discuss → plan → execute → verify → ship` loop is a clean example: each phase is its own slash command, and skipping one by free-form chat means that phase's guard rails never load. Free-form your way through the same task and you slide outside the rails without noticing — the framework's invariants never load, and the session falls back to whatever is in base `CLAUDE.md`. A framework is not a background service; it is a set of entry points you have to actually use.

The shift goes further than framework-reading. Once you know every prompt competes for a shared attention budget, every `CLAUDE.md` line is a per-turn tax, and `/compact` is paraphrase rather than forgiveness, it becomes harder to hand everything to the model and hope — with Claude Code today, with whatever agentic tool replaces it tomorrow. Shallow prompts and vague intent were always bets; you now know the odds. You do not have to stop delegating. You do have to stop pretending the delegation happens in a vacuum. **You are the architect of what the model sees; the model is the mind that works on what you hand it.** The collaboration breaks down exactly when those two roles get confused.

From here, you are reading on your own.

---

## 7. What is next — and what this repo is not

### Coming soon

- [ ] `docs/anti-patterns.md` — the 6 most common mis-placements
- [ ] `docs/lifecycle.md` — when to promote memory → rule, demote CLAUDE.md → rules/
- [ ] `examples/` — one minimal, sanitized example per primitive
- [ ] Case study — one real flow (test authoring) split across all 5 layers, with reasoning

Issues and PRs welcome. If you disagree with a placement, open an issue — the point is to surface the tradeoffs.

### Not in scope

- Not an "awesome-claude" list. Those exist.
- Not a tutorial on installing Claude Code. See [official docs](https://docs.claude.com/en/docs/claude-code).
- Not prompt-engineering tips. Out of scope.
- Not opinions on model choice. Too volatile.

### Call for contributors — a live context dashboard

The central thesis here is that **understanding what Claude knows right now is a skill, and the tools to see it are underbuilt.** `/context` is the best view that exists today, and it is a text dump. There is a buildable gap between it and something genuinely useful — session JSONL already contains ~80% of the data needed to parse, cost-attribute, and visualize every context load in real time. Full RFC, incremental build plan (MVP → TUI → dashboard → team analytics), open design questions, and how to get involved: [`docs/dashboard-rfc.md`](./docs/dashboard-rfc.md).

---

## License

MIT — see [LICENSE](./LICENSE).
