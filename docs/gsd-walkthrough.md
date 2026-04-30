# How GSD actually runs — a concrete walkthrough

> Companion to Sections 5-6 of the main [README](../README.md). This is a case study of *get-shit-done* as installed on a real mid-size Go service, so you can see the primitives from Sections 4-5 in action rather than abstractly.

The project this walkthrough uses is an integration-test framework inside a Vertix monorepo. GSD was running there when I mapped it out — the filenames, agents, and artifacts below are real. Phase numbers and requirement IDs are this project's; the mechanics generalize.

## What GSD installs — authored vs generated state

```
.claude/
  commands/gsd/*.md         81 slash commands — thin dispatchers
  agents/*.md               33 subagents — planner, executor, checker…
  hooks/*.{js,sh}           7 hooks — machine-enforced guards
  get-shit-done/
    workflows/*.md          real orchestration bodies, loaded via @file:
    references/*.md         ~50 reference files (gates, anti-patterns…)
    templates/*.md          scaffolds for generated artifacts
  settings.json             wires hooks to SessionStart / PreToolUse / PostToolUse

.planning/                  ← project state on disk
  config.json               paths, model profile, flags
  PROJECT.md                goals, core value
  REQUIREMENTS.md           requirement IDs (SAFE-01, CORE-01, …)
  ROADMAP.md                phases + success criteria + plan index
  STATE.md                  progress tracker, checkpoint, stopped_at
  research/*.md             domain research
  codebase/*.md             architecture snapshot
  phases/02-framework-infrastructure/
    02-CONTEXT.md           locked decisions (D-01, D-02, …)
    02-PATTERNS.md          existing patterns surveyed
    02-01-PLAN.md … 02-10-PLAN.md  executable plans
    02-DISCUSSION-LOG.md    discuss-phase trace
```

Core distinction: `.claude/` is **authored state** (installed once, never rewritten); `.planning/` is **generated state** (Claude writes into it, then reads it back on the next command).

## What these files actually look like

Excerpts from the same project, showing the *shape* of each file type — not full content.

### Command file — thin dispatcher

`.claude/commands/gsd/plan-phase.md` (whole file ≈ 50 lines):

```markdown
---
name: gsd:plan-phase
description: Create detailed phase plan (PLAN.md) with verification loop
argument-hint: "[phase] [--auto] [--research] [--gaps] [--prd <file>]"
agent: gsd-planner
allowed-tools: [Read, Write, Bash, Glob, Grep, Agent, AskUserQuestion]
---
<objective>
Create executable phase prompts (PLAN.md files) for a roadmap phase.
Orchestrator role: parse arguments, validate phase, research domain, spawn
gsd-planner, verify with gsd-plan-checker, iterate until pass.
</objective>

<execution_context>
@.claude/get-shit-done/workflows/plan-phase.md
@.claude/get-shit-done/references/ui-brand.md
</execution_context>

<process>
Execute the plan-phase workflow from the workflow file end-to-end.
Preserve all gates (validation, research, planning, verification loop).
</process>
```

The dispatcher exists to parse arguments and point Claude at the workflow file. All real logic lives behind `@file:` — loaded only when the command runs.

### Workflow file — orchestration body

`.claude/get-shit-done/workflows/plan-phase.md` (≈ 500 lines — excerpt):

````markdown
<purpose>
Create executable phase prompts. Orchestrates gsd-phase-researcher, gsd-planner,
gsd-plan-checker with a revision loop (max 3 iterations).
</purpose>

<required_reading>
@.claude/get-shit-done/references/revision-loop.md
@.claude/get-shit-done/references/gates.md
@.claude/get-shit-done/references/agent-contracts.md
</required_reading>

<available_agent_types>
- gsd-phase-researcher — Researches technical approaches
- gsd-planner — Creates detailed plans
- gsd-plan-checker — Reviews plan quality before execution
</available_agent_types>

<process>
## 1. Initialize
Load context via gsd-sdk (single query, paths only):
```bash
INIT=$(gsd-sdk query init.plan-phase "$PHASE")
```
Parse JSON for: phase_dir, phase_number, has_research, has_context...

## 2. Parse Arguments
Extract phase number, flags (--research, --gaps, --prd, --text).
````

Step-by-step orchestration with explicit gates and agent-spawn calls. Hits the context window only when `/gsd:plan-phase` actually runs — Axis 2, not Axis 1.

### Agent file — subagent role

`.claude/agents/gsd-planner.md` (≈ 300 lines — excerpt):

```markdown
---
name: gsd-planner
description: Creates executable phase plans with task breakdown, dependency
  analysis, and goal-backward verification. Spawned by /gsd-plan-phase.
tools: Read, Write, Bash, Glob, Grep, WebFetch, mcp__context7__*
---

<role>
You are a GSD planner. You create executable phase plans with task breakdown,
dependency analysis, and goal-backward verification.

Your job: Produce PLAN.md files that Claude executors can implement without
interpretation. Plans are prompts, not documents that become prompts.
</role>

<context_fidelity>
The orchestrator provides user decisions in <user_decisions> tags from
/gsd-discuss-phase. Before creating ANY task, verify:

1. Locked Decisions (## Decisions) — MUST be implemented exactly.
   Reference the decision ID (D-01, D-02) in task actions.
2. Deferred Ideas — MUST NOT appear in plans.
3. Claude's Discretion — Use judgment; document choices in task actions.
</context_fidelity>
```

The description field is the Axis-1 tax (competing with 32 other agents to be picked). The body loads only when `Agent()` spawns this subagent with its isolated context window — that isolation is what lets the planner read 50k tokens of codebase patterns without polluting the parent.

### Reference file — shared vocabulary

`.claude/get-shit-done/references/gates.md` (excerpt):

```markdown
# Gates Taxonomy

Canonical gate types used across GSD workflows.

### Pre-flight Gate
Purpose: Validates preconditions before starting.
Behavior: Blocks entry if conditions unmet. No partial work.
Examples: plan-phase checks REQUIREMENTS.md exists; execute-phase
checks PLAN.md exists.

### Revision Gate
Purpose: Evaluates output quality, routes to revision if insufficient.
Behavior: Loops back to producer with specific feedback. Bounded.
Examples: plan-checker reviewing PLAN.md (max 3 iterations).

### Abort Gate
Purpose: Terminates to prevent damage or waste.
Examples: Context window critically low during execution.
```

References are the shared vocabulary every workflow points to. `plan-phase`, `execute-phase`, and `verify-work` all list `gates.md` in their `<required_reading>` — define once, cite many times.

### Template file — artifact scaffold

`.claude/get-shit-done/templates/summary.md` (excerpt):

```markdown
# Summary Template

Template for .planning/phases/XX-name/{phase}-{plan}-SUMMARY.md.

---
phase: XX-name
plan: YY
subsystem: [auth, payments, ui, api, database, infra, testing, ...]
tags: [searchable tech]
requires:
  - phase: [prior phase]
    provides: [what that phase built]
provides:
  - [bullet list of what this phase delivered]
key-files:
  created: [important files created]
  modified: [important files modified]
key-decisions:
  - "Decision 1"
---
```

Templates define the *shape* of generated artifacts. The executor reads this as the schema to fill in. Not instruction — structural contract.

## What the generated data files look like

### `PROJECT.md` — north star

```markdown
# Integration Test Framework cho middleware/computing

## What This Is
Framework viết và chạy integration test cho toàn bộ biz flow của
middleware/computing. Một lệnh duy nhất để chạy (make itest <flow>)...

## Core Value
Bất kỳ ai — dev mới, dev quen, hay AI agent — chạy integration test cho
1 biz flow chỉ với một lệnh, không phải suy luận setup.

## Requirements
### Validated (brownfield)
- ✓ Go 1.23 + stdlib testing + build tag integration — existing
- ✓ 4 existing *_integration_test.go files — existing

### Active (building toward)
- [ ] Preflight check duy nhất (make itest-check)
- [ ] Một lệnh chạy test: make itest FLOW=<flow>

## Out of Scope
- Auto-refresh session token qua login API
- Auto-tạo test resource nếu thiếu
- Extract framework thành Go module riêng
```

Discuss-phase, plan-phase, and verify-work all read this to stay aligned with what actually matters. If a plan contradicts PROJECT.md, plan-checker flags it.

### `REQUIREMENTS.md` — IDed line items

```markdown
### Safety Rails (SAFE)
- [ ] SAFE-01: .gitignore chứa reports/ và testdata/config.yaml
- [ ] SAFE-03: Trường env: sandbox bắt buộc trong YAML; exit nếu khác
- [ ] SAFE-04: sanitizeForDisk scrub token/authorization/bearer trước ghi disk

### Framework Core (CORE)
- [ ] CORE-01: biz/server/itest_naming.go export ResourceName(flow)
- [ ] CORE-02: biz/server/itest_run.go export TestEnv + NewTestEnv
- [ ] CORE-06: biz/server/itest_preflight.go export CheckPreflight(cfg)

### Concurrent-Agent Safety (CONCUR)
- [ ] CONCUR-01: Hai make itest song song trên 2 agent không xung đột
- [ ] CONCUR-03: GC sentinel file ghi trước mỗi lần tạo resource
```

The atomic commitments. IDs (SAFE-01, CORE-01) become the unit of reference everywhere else — each plan declares which requirement IDs it covers; the verifier checks which IDs actually passed.

### `ROADMAP.md` — phase plan

```markdown
## Phases
- [ ] Phase 1: Safety Rails — gitignore, prod guard, token sanitizer
- [ ] Phase 2: Framework Infrastructure — All six itest_*.go helpers
- [ ] Phase 3: Migration + Spike A — Migrate existing 4 tests to framework
- [ ] Phase 4: Coverage Extension + Spike B — Remaining biz modules

### Phase 2: Framework Infrastructure
Goal: All six itest_*.go helpers exist, TestMain calls preflight, all three
      Makefile targets work
Depends on: Phase 1
Requirements: CORE-01..07, CONCUR-01..03, PFLIGHT-01..03, REPORT-01..04
Success Criteria:
  1. make itest-check with healthy forwards prints [✓] per line, exits 0
  2. make itest FLOW=create_server auto-invokes preflight; fail → exit 2
  3. Two parallel make itest runs produce distinct reports/<ts>-<runID>/

Plans: 10
  - [ ] 02-01-PLAN.md — itest_naming.go: NewRunID + ResourceName
  - [ ] 02-02-PLAN.md — itest_preflight.go: PreflightResult + shell-out check
  - [ ] 02-04-PLAN.md — itest_run.go: TestEnv + NewTestEnv + WriteSentinel
```

Links REQUIREMENTS.md to phases. Success criteria are the verifier's input — planning builds plans to satisfy them, execution implements, verification asserts the criteria hold.

### `STATE.md` — progress tracker

```markdown
---
gsd_state_version: 1.0
milestone: v1.0
status: executing
stopped_at: "Phase 2 planning revision 1 complete — executor TODO: Plan
  02-02 deferred gRPC service probe, revisit after Phase 2 ships"
last_updated: "2026-04-23T04:52:00Z"
progress:
  total_phases: 4
  completed_phases: 1
  total_plans: 14
  completed_plans: 5
  percent: 36
---

## Current Position
Phase: 1 (Safety Rails) — EXECUTING
Plan: 1 of 4
Progress: [████░░░░░░] 36%
```

The session-continuity anchor. The `SessionStart` hook reads this and injects it into every new session — so a fresh Claude opens knowing where the project stopped, not starting from zero.

### `{phase}-CONTEXT.md` — locked decisions

```markdown
# Phase 2: Framework Infrastructure - Context

<decisions>
### Preflight strategy
- D-01: Preflight Go shell-out sang port-forward.sh check qua exec.Command.
  Single source of truth cho port logic.
- D-02: Ngoài port check, preflight Go native check: JWT expiry, env: sandbox,
  resource IDs reachable qua gRPC GET (timeout 3s).
- D-03: Output format per line: [✓] <check> hoặc [✗] <check> — Fix: <cmd>.
  Agent parse bằng prefix.

### GC sentinel system
- D-05: Sentinel file: .gc/<run-id>/<resource-kind>-<id>.json
- D-06: Sentinel viết ngay sau khi gRPC tạo resource thành công
- D-08: Sentinel scope Phase 2: server, volume, port (defer org/keypair to v2)
</decisions>

<deferred>
- SnapshotServerWithVolume — Phase 3 (blocker: resource-side volume getter)
</deferred>
```

The output of discuss-phase. Every downstream agent (planner, researcher, executor) reads this and references decisions by ID (`per D-05`) in their work. This is where "the user decided X" lives durably across compaction and sessions.

### `{phase}-PATTERNS.md` — codebase pattern map

```markdown
## File Classification

| New/Modified File | Role | Closest Analog | Match Quality |
|---|---|---|---|
| itest_naming.go | pure utility | itest_sanitize.go | role-match |
| itest_snapshot.go | test-helper | attach_port_integration_test.go:27 | **exact** |
| cmd/itest-gc/main.go | CLI tool | port-forward.sh:do_check (shell) | role-match (first Go CLI) |

## Pattern Assignments

### itest_naming.go
Analog: biz/server/itest_sanitize.go
Reuse: Package-level pure function, same build-tag, stateless.

Build-tag header pattern (copy verbatim from analog lines 1-4):
```go
//go:build integration
// +build integration

package server
```
```

Output of `gsd-pattern-mapper`. For every new file the planner intends to create, this names an existing analog — "copy this, adapt these three things". Reduces planner freedom (in a good way) and forces consistency with the codebase.

### `{phase}-{n}-PLAN.md` — executable spec

The key file. Excerpt from `02-01-PLAN.md` (full file ≈ 14k chars):

````markdown
---
phase: 02-framework-infrastructure
plan: 01
wave: 1
depends_on: []
files_modified: [biz/server/itest_naming.go, ..._test.go, go.mod]
requirements: [CORE-01, CONCUR-01]
must_haves:
  truths:
    - "Calling NewRunID() twice within the same nanosecond yields distinct strings"
    - "ResourceName(flow) returns string prefixed with itest-<flow>-"
    - "No math/rand import anywhere in the new file"
  artifacts:
    - path: biz/server/itest_naming.go
      provides: NewRunID + ResourceName + runIDSentinelTimeFormat
      contains: "func NewRunID() string"
---

<context>
@.planning/PROJECT.md
@.planning/phases/02-framework-infrastructure/02-CONTEXT.md
@.planning/phases/02-framework-infrastructure/02-PATTERNS.md
@.planning/research/PITFALLS.md

<interfaces>
Expected exported symbols (new file):
```go
// NewRunID returns a unique, sortable identifier for one test run.
// Format: "<unix-seconds>-<uuid[:8]>" e.g. "1745339201-a3f9b2c1".
// Uses crypto/rand via uuid.NewString — NEVER math/rand (Pitfall 12).
func NewRunID() string
```
</interfaces>
</context>

<tasks>
<task type="auto" tdd="true">
  <name>Task 1: Create itest_naming.go with NewRunID + ResourceName</name>
  <read_first>
    - biz/server/itest_sanitize.go (copy build-tag header pattern)
    - 02-PATTERNS.md § "biz/server/itest_naming.go" (lines 28-65)
    - 02-CONTEXT.md § Decisions D-18 (RunID format), D-24 (file size)
  </read_first>
  <behavior>
    - Test 1 (TestNewRunID_Format): regex ^[0-9]{10,}-[0-9a-f]{8}$
    - Test 2 (TestNewRunID_Unique): 1000 calls yield 1000 distinct
    - Test 3 (TestNewRunID_Concurrent): 100 goroutines × 10 → 1000 distinct
    - Test 6 (TestResourceName_NoMathRand): grep -c "math/rand" returns 0
  </behavior>
</task>
</tasks>
````

Not prose — an executable spec. Which files to modify, which requirement IDs it covers, what `must_haves.truths` the verifier will assert, which patterns to copy, exact test cases by name. An executor with this plan does not need to reason about architecture; it implements against a frozen contract. This is why a weaker model can safely run execution.

### `{phase}-{n}-SUMMARY.md` — what got done + deviations

```markdown
---
phase: 01-safety-rails
plan: 01
completed: 2026-04-22
requirements_addressed: [SAFE-01, SAFE-02]
status: complete
---

## What was delivered
- .gitignore extended with 5 new entries (reports/, testdata/yaml, .secrets/*)
- git rm --cached leaked YAML (kept on disk for Plan 01-02 to rewrite)
- .secrets/.gitignore + .secrets/integration.env.example created

## Verification output
$ grep -q "^reports/$" .gitignore; echo $?                → 0
$ git check-ignore testdata/integration_test_config.yaml  → 0 (ignored)
$ git ls-files --error-unmatch ...config.yaml             → 1 (untracked)

## Deviations
- .gitignore pattern for reports/ changed from `reports/` to
  `reports/*` + `!reports/.gitignore` — original pattern ignored the
  placeholder .gitignore itself, making it uncommittable. Fix aligns
  with existing .secrets/* + allow-list pattern.

## Must-haves truths
- [x] .gitignore has all 5 new entries
- [x] Leaked YAML untracked but on disk
- [x] .secrets/ scaffolded, example tracked, real env ignored
```

The closing artifact — what was built, every must-have truth marked ✓/✗, and critically, *deviations* from the plan. Deviations persist: when the same plan is referenced by a later phase, the future reader sees that `reports/` was implemented as `reports/*` + allow-list, not as the original spec said. This is how the framework accumulates institutional memory on disk across sessions.

## Milestone-level meta-loop

```
/gsd:new-project    →  scaffolds PROJECT.md, REQUIREMENTS.md, ROADMAP.md, STATE.md
/gsd:new-milestone  →  scopes a milestone inside the roadmap
      │
      ▼
   [phase loop]      ←  iterate per phase
      │
      ▼
/gsd:ship            →  push branch, open PR, merge
```

## One phase loop, end-to-end — Phase 2 as example

### Step 1: `/gsd:discuss-phase 2`

The slash command (15-line frontmatter) loads `workflows/discuss-phase.md` via `@file:`. The workflow orchestrates:

1. Read `PROJECT.md`, `REQUIREMENTS.md`, `STATE.md`, every prior `*-CONTEXT.md` → build context for the model.
2. Spawn `gsd-assumptions-analyzer` to scan the codebase for gray areas.
3. Present gray areas via `AskUserQuestion` (TUI menu).
4. For each selected area, optionally spawn `gsd-advisor-researcher` to compare options.
5. Lock decisions in `D-01: …` format.

**Generated artifact:** `.planning/phases/02-framework-infrastructure/02-CONTEXT.md` — decisions D-01 through D-N with rationale.

This is what Claude in a later turn *cannot* see in message history (gone after `/compact`) but *can* see on disk.

### Step 2: `/gsd:plan-phase 2`

Orchestrator workflow runs three agents in a revision loop (max 3 iterations):

```
┌─────────────────────────────────────────────────────┐
│ 1. gsd-phase-researcher                             │
│    input: CONTEXT.md, PROJECT.md, codebase/*.md     │
│    output: 02-RESEARCH.md (domain knowledge)        │
├─────────────────────────────────────────────────────┤
│ 2. gsd-pattern-mapper                               │
│    input: codebase + CONTEXT.md                     │
│    output: 02-PATTERNS.md (existing patterns)       │
├─────────────────────────────────────────────────────┤
│ 3. gsd-planner                                      │
│    input: CONTEXT.md + RESEARCH.md + PATTERNS.md    │
│    output: 02-01-PLAN.md … 02-N-PLAN.md             │
├─────────────────────────────────────────────────────┤
│ 4. gsd-plan-checker                                 │
│    input: plans + CONTEXT.md                        │
│    output: pass / fail + feedback                   │
│    fail → loop back to gsd-planner with feedback    │
│    max 3 revisions                                  │
└─────────────────────────────────────────────────────┘
```

Each generated plan carries the structured frontmatter shown earlier (see §*`{phase}-{n}-PLAN.md` — executable spec*) — `must_haves.truths` are the assertions the verifier will check in Step 4.

### Step 3: `/gsd:execute-phase 2`

Unlike the prior two, the orchestrator here deliberately **stays lean** (~15% context budget); all real execution happens in fresh-context subagents. The workflow:

1. Read every `02-*-PLAN.md`, parse `depends_on` → build a dependency graph.
2. Group into waves (plans with no mutual dependencies → same wave, run in parallel).

   For Phase 2: Wave 1 = `02-01-PLAN.md` (naming, depends on nothing); Wave 2 = plans that depend on naming, run in parallel.

3. For each plan in a wave, spawn a `gsd-executor` subagent with:
   - An `<execution_context>` block naming only the files it needs to read (its PLAN.md + `templates/summary.md`).
   - A prompt that does NOT include other plan files — each executor only sees its own plan.
4. The executor reads its PLAN.md, implements files listed in `files_modified`, writes `02-01-SUMMARY.md`.
5. The orchestrator collects summaries, updates `STATE.md`.

**This is where model downshift is genuinely safe.** The executor sees a plan that has already been checker-verified, with explicit `must_haves`, scope bounded by `files_modified`. Running Haiku as the executor does not drift, because the plan is the guardrail.

### Step 4: `/gsd:verify-work 2`

Spawns `gsd-verifier` to read `02-*-PLAN.md` alongside the actual code and assert each line under `must_haves.truths` is true. Output `02-UAT.md`. If it fails, `/gsd:plan-phase 2 --gaps` produces a fix plan and the loop returns to Step 3.

### Step 5: `/gsd:ship 2`

Pushes the branch, opens a PR with a body auto-generated from every SUMMARY in the phase.

## State flow on disk

```
turn 1:   /gsd:discuss-phase 2    → writes 02-CONTEXT.md
turns 2-5 (model compacted)         message history rewritten
turn 6:   /gsd:plan-phase 2        → READS 02-CONTEXT.md from disk
                                     → writes 02-RESEARCH.md, 02-PATTERNS.md, 02-*-PLAN.md
turns 7-15 (model compacted)        message history rewritten
turn 16:  /gsd:execute-phase 2     → READS 02-01-PLAN.md from disk
                                     → writes code + 02-*-SUMMARY.md
turns 17-30 (fresh session!)        zero message history
turn 31:  /gsd:verify-work 2       → READS PLAN.md + code
                                     → writes 02-UAT.md
```

At every step, what Claude needs is on disk at a convention path. No step depends on "does Claude remember the prior session".

## Hooks running underneath

In parallel with every command above, hooks enforce at PreToolUse / PostToolUse:

| Hook | When | What it does |
|---|---|---|
| `gsd-prompt-guard.js` | PreToolUse Write/Edit | Blocks prompt injection from files Claude read |
| `gsd-read-guard.js` | PreToolUse Write/Edit | Requires Read before Edit on the same file |
| `gsd-workflow-guard.js` | PreToolUse Write/Edit | Blocks writes into a phase marked "completed" in STATE.md |
| `gsd-read-injection-scanner.js` | PostToolUse Read | Scans just-read content for injection |
| `gsd-phase-boundary.sh` | PostToolUse Write/Edit | Logs writes into `.planning/` for tracking |
| `gsd-validate-commit.sh` | PreToolUse Bash | Enforces Conventional Commits on `git commit` |
| `gsd-context-monitor.js` | PostToolUse Bash/Edit/Write/Agent | Tracks context budget, warns near `/compact` |
| `gsd-session-state.sh` | SessionStart | Injects `STATE.md` into the opening context of every new session |

The last one is worth calling out: the `SessionStart` inject means a fresh session opens with Claude already knowing *which phase is active, which plan just finished* — no user re-briefing needed. That is Axis 3 doing part of what message history used to do.

## Injecting a rule into GSD — where, when, by whom

Given how tight GSD's machinery is, a fair question is: *when I have a new invariant (e.g., "integration test code must access resources through the manager layer, not directly through adapters"), how do I actually get it into the system?* This section compares what that looks like with vs. without a framework, and clarifies the boundary GSD enforces between authored and generated state.

### The baseline: vibe / own-framework

Without a framework, rule-emergence typically looks like this:

1. You notice Claude doing something wrong.
2. You type *"don't do that, do X instead"*.
3. Claude acknowledges, fixes the current instance.
4. **End.** Nothing durable. Next session resets; Claude re-does the wrong thing.

Base Claude Code already gives you the primitives for durability — `CLAUDE.md`, `.claude/rules/*.md`, hooks. The primitives are not missing. What is missing is a **natural moment** to promote an in-session observation into a durable rule. Stopping to author a rule file feels like friction mid-work, so most users either dump a bullet into `CLAUDE.md` (easy, expensive long-term — the graveyard pattern from README Section 1) or skip it entirely and hope memory holds.

If you are building your own framework, you can design that moment into your loop — an end-of-session checklist, a "lessons to codify" ritual. But nothing enforces that you actually use it.

### Injection points GSD adds

GSD does not replace the primitives. It adds **scheduled authoring moments** where writing a rule is the natural next step rather than an interruption.

| Injection point | When it shows up | Scope | Cost |
|---|---|---|---|
| `.claude/rules/*.md` path-scoped | Anytime | Files matching `paths:` | Axis 2 on file read |
| `PROJECT.md` Constraints | `/gsd:new-project` or manual edit | Whole project | Axis 1 (PROJECT.md re-read by many workflows) |
| `REQUIREMENTS.md` new ID | `/gsd:new-project`, `/gsd:new-milestone` | Whole project, must-be-verified | Axis 1 |
| Phase `CONTEXT.md` as `D-XX` | `/gsd:discuss-phase` | Phase-scoped | Axis 2 on command invoke |
| `CLAUDE.md` | Anytime | Every turn | Axis 1 (expensive) |
| Hook (`.claude/hooks/`) | Anytime | Deterministic | Axis 3 (outside the model) |
| `LEARNINGS.md` via `/gsd:extract-learnings` | After a phase | Documentation only | Axis 2 |
| `seeds/SEED-*.md` via `/gsd:plant-seed` | When you notice "future rule material" | Surfaces at new milestone | Axis 2 |

The first six rows are base Claude Code plus GSD's own state files — authoring into them is the same as in vibe mode; GSD just reuses them. The last two (`extract-learnings`, `plant-seed`) are GSD-specific moments:

- `/gsd:new-project` explicitly asks for Constraints → you put your rule reference there instead of trying to remember later.
- `/gsd:discuss-phase` explicitly asks about gray areas → you lock a rule as `D-XX` in the same conversation where you would have lost it otherwise.
- `/gsd:extract-learnings N` explicitly prompts reflection after a phase — *"what did this phase teach us that should persist?"*
- `/gsd:plant-seed` explicitly captures ideas that are not ready to become rules yet.

None of these force authoring, but each is a point where the user would have otherwise had to interrupt their own flow. That is the real affordance GSD adds for rule authoring: **nothing you could not do without it, but the interruption cost is absorbed by the workflow.**

### Does GSD ever generate a rule automatically?

**No.** GSD never writes into `.claude/rules/`. The files it generates all stay inside `.planning/`:

- `LEARNINGS.md` — post-phase reflection
- `seeds/SEED-*.md` — future-idea capture
- `intel/*` — codebase intelligence snapshots
- `CONTEXT.md` — phase-scoped decisions

`.claude/rules/` is **authored state**. GSD respects the authored vs. generated boundary — everything GSD writes is generated; everything humans write is authored.

This is a deliberate design choice, not an oversight. Three reasons:

1. **Auto-promotion would build a graveyard.** Every edge case in `LEARNINGS.md` becoming a permanent rule is exactly the failure mode README Section 1 warns against. Rules survive forever as Axis 1 tax; one-off incidents do not belong there.
2. **Promotion is a judgment call.** The right gate is *"will this recur?"* (README Section 6). Only humans can answer that; GSD can surface the candidate but cannot decide.
3. **Authored state must stay stable to be committable.** If GSD rewrote `.claude/` each run, the directory would drift per session and could not be reviewed in PR. Keeping `.claude/` human-only preserves it as reviewable config.

### Promotion workflow in practice

The actual path from "Claude made a mistake" to "Claude never makes that mistake again" runs through human judgment, with GSD providing data-gathering on either side:

```
Phase N executing
  → Claude does something wrong
  → You correct in-session (ephemeral fix)
  → Phase N completes

/gsd:extract-learnings N
  → writes .planning/LEARNINGS.md with patterns / surprises / decisions

[you read LEARNINGS.md]
  → entry: "Plan N-03 needed two revisions because executor called adapter directly"
  → judgment: this recurs whenever someone touches test code → invariant

[you manually author .claude/rules/integration-test-access-pattern.md]
[you manually add 1 line to PROJECT.md Constraints]

/gsd:plan-phase N+1
  → planner picks up the new rule automatically
  → translates into must_haves.truths in the next phase's plans
  → verifier asserts on it going forward
```

GSD's role is **signal, not promotion.** `LEARNINGS.md` and `seeds/` are data you use to make the promotion decision; the decision itself is yours.

### Four non-obvious insights about rule authoring

Four second-order observations that are easy to miss when you first adopt GSD.

**1. The authoring moment encodes rule quality.**

Writing a rule in three different cognitive states produces three different kinds of rule:

- **Mid-execution (when Claude just did it wrong)** — reactive, emotional, usually too narrow or too broad. *"Never call the adapter directly"* because you just saw one misuse — but there may be legitimate use cases you have not considered.
- **During discuss-phase (forward reasoning)** — medium quality. You are guessing about what might recur, without evidence.
- **During extract-learnings (retrospection, after the phase)** — highest quality. You have the concrete incident, you have data on recurrence, you are out of heat-of-battle.

GSD does not place `/gsd:extract-learnings` after each phase by accident — that is the optimal cognitive state for rule authoring. You are not debugging, not on deadline, you are reflecting. That is when a rule has its best chance of being right.

Operational consequence: when you notice something rule-worthy mid-execution, **do not author immediately**. Jot it to a scratch note, wait for extract-learnings. Rules written under stress almost always need a rewrite.

**2. `.claude/rules/` is only half the rule surface.**

The other half is `must_haves.truths` inside each PLAN.md. They have genuinely different properties:

| | `.claude/rules/*.md` | `must_haves.truths` in PLAN.md |
|---|---|---|
| Author | Human | Planner (AI) |
| Scope | Long-term, cross-phase | Short-term, one plan |
| Lifecycle | Forever | Effectively dies after verification |
| Enforcement | Indirect (Claude reads) | Direct (verifier asserts) |
| Cost to create | High (stop, think, commit) | Zero (planner generates) |

Most teams default to routing every invariant into `.claude/rules/`. That is a mistake: **phase-specific invariants (acceptance criteria) belong in must_haves, not rules.** For example, *"TestEnv.RefreshServer must call manager.GetServerByID"* is a must_have for Plan 02-04, not a project-wide rule. Routing it to rules pays an Axis-2 cost every time another test file is read; leaving it in must_haves verifies once and is done.

Conversely: *"No test code may import adapter packages"* is a cross-phase architectural invariant — deserves to be a rule. The distinction to hold:

- **Cross-phase + architectural → `.claude/rules/`**
- **Phase-specific + acceptance criterion → `must_haves.truths`**

**3. Rules age silently — the biggest gap in GSD's rule model.**

GSD has commands for almost everything: `extract-learnings`, `audit-fix`, `stats`, `health`. It does not have `audit-rules`. There is no mechanism to prompt *"is rule X, written six months ago, still correct?"*

The asymmetry is striking:

| Generated state | Lifecycle tracking |
|---|---|
| `STATE.md` | `stopped_at`, `last_updated` |
| `SEED-*.md` | trigger conditions surface at new milestone |
| `LEARNINGS.md` | per-phase, timestamped |

| Authored state | Lifecycle tracking |
|---|---|
| `.claude/rules/*.md` | **None** |
| `CLAUDE.md` | **None** |

A rule from Phase 1 may have gone stale by Phase 7 when the architecture changed, but nothing prompts you to check. Claude will keep reading and obeying, including obeying constraints that no longer apply — and you will see output that looks wrong without understanding why.

Workaround: schedule rule review yourself after each milestone. Open every file in `.claude/rules/`, ask *"does this still recur? is the reason still valid?"*. Whatever does not survive the question, delete. **GSD does not do this for you. It is discipline you must schedule yourself.**

**4. You are the primary reader of rules, not Claude.**

Imagine a rule that lives for a year:

- Claude reads it: ~1000 turn-loads (path-scoped, whenever a matching file comes into scope)
- You read it: 20-50 times — PR reviews, teammate onboarding, debugging drift, editing neighbouring rules

Claude reads at surface depth (pattern match). You read at deep depth (understand, decide, modify). **You are the low-throughput, high-stakes reader.**

The consequence for how to write rules:

- Prose, context, and *why* matter more than MUST / MUST NOT bullets. Claude parses both fine; you-in-six-months parses prose better than cryptic bullets.
- Include incident links or commit hashes as grounding. *"SpiceDB auth check bypassed on the adapter path — see commit abc123"* has retrospective value for you that Claude does not need.
- Write the way you would write documentation with teeth, not the way you would write instructions for an AI.

This inverts a common assumption: *rule = Claude training material*. It is not. **Rule = documentation that Claude happens to also read.** You remain the primary consumer.

---

Taken together: GSD teaches you **when** to promote (after retrospection, not in the middle of battle) and **where** to promote (rules vs. must_haves, by scope). It does not teach you **when to demote** (when a rule has stopped being true). That discipline remains yours. The framework opens the door; you still have to walk through.

### Summary — vibe vs. GSD on rule authoring

| | Vibe / own-framework | GSD |
|---|---|---|
| Primitives for injection | Complete (CLAUDE.md, rules/, hook) | Same + CONTEXT.md, REQUIREMENTS.md, seeds |
| Natural authoring moments | None — you create them | discuss-phase, extract-learnings, plant-seed |
| Gate between observation and rule | Entirely your discipline | LEARNINGS.md gathers data; you still judge |
| Auto-generation of rules | No | No (design choice) |

The tooling for creating a rule is identical in both cases. What GSD adds is **rhythm** — scheduled pauses where authoring is the natural next step. Someone who already has the habit of promoting observations to rules (the habit discussed in README Section 5) gets that habit accelerated by GSD. Someone without the habit gets prompted at the right moments but is not forced. The framework speeds up discipline that exists; it does not manufacture it.

## The mechanics, rolled up

| Layer | README axis | Role |
|---|---|---|
| 81 slash commands | Axis 1 (~20 lines description each) | Explicit entry points |
| ~50 reference files | Axis 2 | Loaded when a workflow needs them |
| 33 subagents | Isolated context window | Primary unit of work |
| `.planning/*.md` | Disk — outlives `/compact` | State across sessions |
| 7 hooks | Axis 3 (outside the model) | Enforce invariants, inject state |
| `CLAUDE.md` + `.claude/rules/` | Axis 1 | Project-specific invariants you add on top |

What makes GSD *work* is not the model's raw intelligence — it is these four layers interlocking: **command dispatch → workflow orchestration → subagent execution → hook enforcement**, with `.planning/` as the state bus between them all.
