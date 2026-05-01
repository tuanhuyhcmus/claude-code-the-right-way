# POC: pair Claude Code with a "not-yet-strong" model via Ollama

> Companion to [Part 1 of the README](../README.md) — the line *"Try pairing a strong client with a not-yet-strong model (e.g. models under 30B parameters): no matter how good the context is, it will not behave in a way you find satisfying."*
>
> This POC is the sibling of [client → proxy → server](./poc-client-proxy-server.md). That one lets you see **what Claude Code sends**; this one lets you feel **what Claude Code is supposed to receive**. Once you run it yourself, you will understand why the README keeps pointing at the model — and why "give it more context" cannot save you when the server side is weak.

The setup has two pieces:

- **Ollama** — runs locally, binds `:11434`. Since `0.14` (early 2026), Ollama ships with a built-in **Anthropic Messages API compatibility layer** on that same endpoint, so **no translation proxy is needed**.
- **Claude Code CLI** — point `ANTHROPIC_BASE_URL` at Ollama, pick a model you have pulled.

```
Claude Code ──► localhost:11434 (Ollama, Anthropic-compatible) ──► gemma3:4b (local)
```

Compared to the other POC (CLIProxyAPI + mitm), this is **much simpler** — no proxy, no OAuth — because the goal is to feel "what does a weak model feel like," not to capture requests.

---

## 1. Install Ollama

macOS (Homebrew):

```bash
brew install ollama
```

Linux:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Verify:

```bash
ollama --version       # must be >= 0.14 for the Anthropic-compat layer
```

Start the daemon (keep it running in its own terminal, or use `brew services start ollama`):

```bash
ollama serve
```

> By default Ollama binds `127.0.0.1:11434`. To expose on the LAN: `OLLAMA_HOST=0.0.0.0:11434 ollama serve`.

## 2. Pull a "not-yet-strong" model

The point of the POC is to pick a model **just capable enough to do tool calling, just weak enough that you feel its limits**. Current Gemma 3 tags:

| Tag           | Size  | Context |
| ------------- | ----- | ------- |
| `gemma3:270m` | 292MB | 32K     |
| `gemma3:1b`   | 815MB | 32K     |
| `gemma3:4b`   | 3.3GB | 128K    |
| `gemma3:12b`  | 8.1GB | 128K    |
| `gemma3:27b`  | 17GB  | 128K    |

For the exercise:

- **`gemma3:4b`** — the sweet spot. 128K context (enough for a real Claude Code session), modest size, runs on a 16GB-RAM laptop. This is roughly what the README means by "under 30B."
- `gemma3:12b` if your machine has the headroom — still well below frontier, enough to feel the gap.
- **Avoid** `gemma3:1b` or `:270m` for this POC — their 32K context dies the moment Claude Code injects its system prompt + tools array on the first turn. You'll only see context-overflow errors, not the thing you came to feel.

```bash
ollama pull gemma3:4b
```

Smoke-test through Ollama's native endpoint:

```bash
ollama run gemma3:4b "write a fibonacci function in Go"
```

If you see code, you're good.

## 3. Point Claude Code at Ollama

Two ways, same result.

### 3.1. The easy way — `ollama launch`

Ollama 0.14+ ships a wrapper that exports the env vars and launches claude:

```bash
ollama launch claude --model gemma3:4b
```

### 3.2. The manual way — export env yourself

```bash
export ANTHROPIC_BASE_URL=http://localhost:11434
export ANTHROPIC_AUTH_TOKEN=ollama
export ANTHROPIC_API_KEY=""

claude --model gemma3:4b
```

> Some `claude` CLI builds skip an empty `ANTHROPIC_API_KEY` and only read `ANTHROPIC_AUTH_TOKEN`. If you hit `Missing API key`, also set `ANTHROPIC_API_KEY=ollama`.

## 4. What to feel

This is the part of the POC that actually pays off. Open a fresh Claude Code session and try the same routine tasks you'd do with Sonnet/Opus, **side by side**:

- **First-turn latency** — `gemma3:4b` on CPU/Apple Silicon usually takes a few seconds for the first turn. Sonnet replies almost instantly. That's your first realization: speed is a UX dimension, not just a benchmark number.
- **Tool calling** — ask Claude `read README.md and summarize`. Watch:
  - Does it call the right `Read` tool?
  - Does it hallucinate tools that don't exist, or call existing ones with wrong arguments?
  - Does it summarize accurately, or invent content?
- **Multi-step plan** — give it something that needs 3-4 chained tools (e.g. `find all .go files containing TODO, read the first three, propose fixes`). Watch whether it stops mid-way, repeats itself, or veers off after step 2.
- **Long context** — paste 5 source files into the prompt and ask a question that requires synthesizing across them. This is where Sonnet/Opus shine and `gemma3:4b` falls hardest — not because the window is small (gemma3:4b has 128K), but because of **self-attention dilution** ([deep dive in `docs/llm-mechanics/self-attention.md`](./llm-mechanics/self-attention.md)).
- **Behaviour with CLAUDE.md / skills** — if the repo has a `CLAUDE.md` and rules, weak models tend to ignore or misread them. That is a direct demonstration of the README's running thesis: **a strong client and a clean context are useless without enough reasoning capacity on the model side**.

For a closer look, **combine this with the other POC**: put mitmweb between Claude Code and Ollama:

```bash
docker run --rm -it -p 8080:8080 -p 8082:8081 \
  mitmproxy/mitmproxy mitmweb \
    --mode reverse:http://host.docker.internal:11434 \
    --web-host 0.0.0.0 \
    --set web_open_browser=false \
    --set web_password=admin

export ANTHROPIC_BASE_URL=http://localhost:8080   # via mitm instead of :11434
claude --model gemma3:4b
```

You'll see the **exact same** system prompt + tools array that Sonnet handles cleanly produce a misaligned tool call from `gemma3:4b` — the most visceral proof that "a strong client cannot rescue a weak model."

## 5. Cleanup

```bash
unset ANTHROPIC_BASE_URL ANTHROPIC_AUTH_TOKEN ANTHROPIC_API_KEY

# Stop Ollama
brew services stop ollama       # macOS
# or Ctrl-C the terminal running `ollama serve`

# Free up disk if you're done
ollama rm gemma3:4b
```

---

## Troubleshooting

| Symptom                              | Common cause                                                                                                              |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| `connection refused` from Claude     | Ollama not serving. Verify: `curl http://localhost:11434/api/tags`.                                                       |
| `model not found`                    | You haven't `ollama pull <tag>`. List installed: `ollama list`.                                                           |
| `404` or Anthropic-format error      | Ollama < 0.14 has no Anthropic-compat. Upgrade: `brew upgrade ollama`, or rerun `install.sh`.                             |
| Claude hangs / times out             | Model too big for available RAM, Ollama is swapping. Drop to a smaller tag (`gemma3:4b` instead of `:12b`).               |
| Tool calls keep going wrong          | This is the small-model signature. It is *what the POC is for*, not a bug to fix.                                         |
| `context length exceeded`            | You picked `gemma3:1b`/`:270m` (32K). Move to `:4b` (128K).                                                               |
| `Missing API key`                    | Also set `export ANTHROPIC_API_KEY=ollama` (some `claude` builds reject an empty value).                                  |
| Replies suspiciously good            | You may have routed through `kimi-k2.5:cloud` / `glm-5:cloud` (Ollama Cloud, not local). Check `ollama list` to confirm.  |

---

## Notes

- The pattern isn't tied to Gemma. Swap in `qwen3:4b`, `llama3.2:3b`, `phi3:3.8b`... the conclusion is the same: **a tool-using agent + small local model = a noticeably worse experience than the same client with a frontier model**.
- "Local" doesn't mean "free": it costs RAM, battery, and time. If your goal is *real work*, the takeaway is usually "go back to Sonnet/Opus." If your goal is *learning*, this is one of the cheapest ways to feel the ceiling.
- Don't use this POC to grade Gemma 3 — Gemma 3 may be excellent on chat, summarization, or classification tasks. The exercise here only shows it isn't yet enough for the **tool-using, long-horizon agent** Claude Code is designed for.

## Sources

- [Ollama: Claude Code integration docs](https://docs.ollama.com/integrations/claude-code)
- [Ollama Blog: Claude Code with Anthropic API compatibility](https://ollama.com/blog/claude)
- [Ollama Library: Gemma 3](https://ollama.com/library/gemma3)
- [Anthropic API: ANTHROPIC_BASE_URL env](https://docs.anthropic.com/en/api/getting-started)
