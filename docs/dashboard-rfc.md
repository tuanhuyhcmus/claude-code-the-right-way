# Building the missing dashboard — a call for contributors

*[English] · [Tiếng Việt](./dashboard-rfc.vi.md)*

*RFC for a live context-inspector tool. Moved out of the main README because it is a side-quest to the main argument — but an actionable one.*

The central thesis of the main README is that **understanding what Claude knows right now is a skill, and the tools to see it are underbuilt.** The built-in `/context` command is the best diagnostic view that exists, and it is a text dump. Anthropic's interactive ["Explore the context window"](https://code.claude.com/docs/en/context-window#what-the-timeline-shows) is a hardcoded simulation in their docs — not a live inspector of your own session. The closest open-source attempts fall short in specific, addressable ways: [`claude-devtools`](https://github.com/matt1398/claude-devtools) aggregates skills into a single bucket; [`token-dashboard`](https://github.com/nateherkai/token-dashboard) tracks invoked skills only, not descriptions; [`cc-dump`](https://github.com/brandon-fryslie/cc-dump) shows raw system prompt but does not parse it.

The gap is real, and — after verifying it by reading our own session JSONL during the writing of this repo — it is **buildable**. Three facts we confirmed:

1. **Session JSONL contains roughly 80% of the data.** Every `~/.claude/projects/<project>/<session-id>.jsonl` contains structured attachments (`skill_listing`, `nested_memory`, `deferred_tools_delta`, `hook_success`, and ~30 other types) with the actual text of every skill description, rule, nested `CLAUDE.md`, and MCP tool name that entered context. Every assistant message records exact per-turn `cache_creation_input_tokens`, `cache_read_input_tokens`, and `ephemeral_5m` / `ephemeral_1h` breakdowns.
2. **What is missing is small and addressable.** The system prompt text is not in JSONL, but it is in [`Piebald-AI/claude-code-system-prompts`](https://github.com/Piebald-AI/claude-code-system-prompts), updated per Claude Code version. MCP tool full schemas are not in JSONL either, but `/mcp` and the MCP server can provide them. Per-token attribution is a call to Anthropic's free [`count_tokens` endpoint](https://platform.claude.com/docs/en/api/messages-count-tokens).
3. **No existing tool stitches these together at `/context` granularity**, let alone over time, across a team, or with cache-lifecycle visualization.

## An incremental build plan

| Stage | Scope | Effort | Who it is for |
| :---- | :---- | :----- | :------------ |
| **MVP** — CLI snapshot | Read a JSONL file, produce static per-skill / per-rule / per-tool token breakdown matching `/context` output. Tokenize via `count_tokens`, cache locally. | ~1 week, solo | Individual engineers who want to audit their own setup |
| **V2** — live TUI | Watch the JSONL being written in an active session (`fswatch` / `inotify`), update a terminal UI with category bars + detail panel. | +2–3 weeks | Daily Claude Code users, power users |
| **V3** — web dashboard | Backend serving parsed state over WebSocket; React/Svelte timeline with drilldown inspired by Anthropic's simulation; subagent sidechain parsing; `/compact` before/after comparison; cache hit/miss timeline. | +1–2 months, 2 contributors | Teams, PR demos, conference talks |
| **V4** — team analytics | Multi-user session aggregation, cost alerts, anomaly detection, privacy-preserving mode that strips conversation content. | +1–3 months, 3–4 contributors | Eng leadership, finance, platform teams |

Each stage is independently useful. The MVP alone beats every existing open-source alternative.

## Open design questions

- **License.** MIT, Apache, or source-available with commercial restriction?
- **Scope.** Should the parser also handle other agentic CLIs with similar JSONL layouts (OpenCode, Cline, Codex, OpenClaw), or stay Claude-Code-focused?
- **Monetization.** Open-source core with a paid team tier, or fully OSS and rely on sponsorship?
- **Hosting.** Self-hosted only, or a hosted SaaS option for the team dashboard?
- **Schema stability.** Claude Code JSONL format can change without notice. Do we pin to a version matrix and integration-test, or parse defensively and warn on unknowns?

## How to get involved

- Open an issue on the main repo with your take on scope or a tier you want to own.
- If you have already built something adjacent ([`ccusage`](https://github.com/ryoppippi/ccusage), [`token-dashboard`](https://github.com/nateherkai/token-dashboard), [`claude-devtools`](https://github.com/matt1398/claude-devtools), [`cc-dump`](https://github.com/brandon-fryslie/cc-dump), [`proxyclawd`](https://github.com/dyshay/proxyclawd)) and want to collaborate rather than compete, say so — there is more gap than there are people trying to fill it.

The fastest path is probably: one contributor on the JSONL-parser core, one on tokenization and cost calculation, one on UI. Three people, three months, a tool nobody else has built.
