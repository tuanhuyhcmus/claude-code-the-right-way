# POC: client → proxy → server — see what Claude Code actually sends

> Companion to [Part 1 of the README](../README.md) — the paragraph that says *"setting up your own POC: client (Claude Code) → proxy (any one will do) → server"*.
>
> The point of this exercise is not to install more tools. It is to **look at** what Claude Code really sends: the system prompt it injects on your behalf, the `tools` array it declares, the entire message history it replays every turn, the actual size of a context window that *feels* like "a few chat lines." Most of the intuition about context and Claude Code in the README will arrive on its own once you've watched a handful of flows go by.

The setup is three components, each in its own container:

- **CLIProxyAPI** (`:8317`) — the **server**. It wraps Claude Code/Pro/Max OAuth into an Anthropic-compatible endpoint (`POST /v1/messages`). Claude Code therefore "thinks" it is talking to `api.anthropic.com`.
- **mitmweb** (`:8080` for traffic, `:8082` for the UI) — the **proxy**. Reverse mode: clients hit `:8080`, mitm logs the flow, then forwards to `:8317`.
- **Claude Code CLI** — the **client**. We point `ANTHROPIC_BASE_URL` at mitm so every request goes through it.

```
Claude Code ──► localhost:8080 (mitmweb)  ──►  host.docker.internal:8317 (CLIProxyAPI)  ──►  api.anthropic.com (OAuth)
                       │
                       └──► UI: http://localhost:8082
```

> Why CLIProxyAPI, instead of pointing Claude Code → mitm → `api.anthropic.com` directly? Claude Code authenticates with OAuth (Pro/Max), not a plain `x-api-key`. CLIProxyAPI holds the OAuth token and rewrites the auth header before calling upstream — you don't have to fight auth, you can focus on watching the request body. If you use a paid API key instead of a subscription, you can drop CLIProxyAPI and have mitm reverse-proxy `api.anthropic.com` directly.

---

## 1. Start CLIProxyAPI (server)

> Repo: <https://github.com/router-for-me/CLIProxyAPI> — prebuilt image: `eceasy/cli-proxy-api:latest`.

### 1.1. Prepare directory & config

```bash
mkdir -p ~/cliproxy/auths
cd ~/cliproxy
```

Minimal `config.yaml`:

```yaml
host: ""           # bind all interfaces; use "127.0.0.1" to restrict to local
port: 8317

auth-dir: "/root/.cli-proxy-api"

# API key clients hit. Claude Code will pass it as ANTHROPIC_API_KEY.
# Any value works — it sits behind mitm and is never exposed externally.
api-keys:
  - "proxy"

debug: true
logging-to-file: false

# Round-robin if you log in multiple accounts later
routing:
  strategy: "round-robin"
  session-affinity: false
```

### 1.2. Log in to Claude (one-time, OAuth)

Login opens a browser on the host and uses port `54545` for the callback. Since we run inside Docker, expose that port:

```bash
docker run --rm -it \
  -p 54545:54545 \
  -v "$PWD/config.yaml:/CLIProxyAPI/config.yaml" \
  -v "$PWD/auths:/root/.cli-proxy-api" \
  eceasy/cli-proxy-api:latest \
  /CLIProxyAPI/CLIProxyAPI --claude-login
```

- The container prints a `https://claude.ai/...` URL → open it on the host browser, log in with your Claude account (Pro/Max).
- After consent, the browser hits `http://localhost:54545/...`, the container writes the token into `~/cliproxy/auths/`, then exits.
- Verify: `ls ~/cliproxy/auths/` should show JSON token files.

> Headless box (no browser): add `-no-browser` so the CLI prints the URL — you complete the flow in another machine's browser.

### 1.3. Run CLIProxyAPI in server mode

Terminal 1 (keep it running):

```bash
docker run --rm -it \
  --name cliproxy \
  -p 8317:8317 \
  -v "$PWD/config.yaml:/CLIProxyAPI/config.yaml" \
  -v "$PWD/auths:/root/.cli-proxy-api" \
  eceasy/cli-proxy-api:latest
```

Smoke-test directly (without mitm):

```bash
curl -s http://localhost:8317/v1/models -H "x-api-key: proxy" | jq .
```

If you see a model list, the server is up.

### 1.4. (Optional) Log in to other providers

CLIProxyAPI also wraps Codex / Gemini, useful if you want to compare an OpenAI Codex or Gemini CLI client through the same proxy pattern:

```bash
# Codex (OpenAI subscription)
docker run --rm -it -p 1455:1455 \
  -v "$PWD/config.yaml:/CLIProxyAPI/config.yaml" \
  -v "$PWD/auths:/root/.cli-proxy-api" \
  eceasy/cli-proxy-api:latest \
  /CLIProxyAPI/CLIProxyAPI --codex-login

# Gemini
docker run --rm -it -p 8085:8085 \
  -v "$PWD/config.yaml:/CLIProxyAPI/config.yaml" \
  -v "$PWD/auths:/root/.cli-proxy-api" \
  eceasy/cli-proxy-api:latest \
  /CLIProxyAPI/CLIProxyAPI --login
```

---

## 2. Start mitmweb (proxy)

Terminal 2:

```bash
docker run --rm -it \
  -p 8080:8080 \
  -p 8082:8081 \
  mitmproxy/mitmproxy \
  mitmweb \
    --mode reverse:http://host.docker.internal:8317 \
    --web-host 0.0.0.0 \
    --set web_open_browser=false \
    --set web_password=admin
```

- `8080` → the port clients (Claude Code) hit.
- `8082` → mitmweb's web UI (internal mapping `8081`).
- `--mode reverse:...` → forward to CLIProxyAPI on the host.
- `--set web_password=admin` → UI login password.

Open the UI: <http://localhost:8082> (password `admin`).

---

## 3. Run Claude Code through the proxy (client)

Terminal 3:

```bash
export ANTHROPIC_BASE_URL=http://localhost:8080
export ANTHROPIC_API_KEY="proxy"        # must match api-keys in config.yaml
claude --model opus-4-6
```

Every `POST /v1/messages` request from Claude Code now goes through:

1. → mitmweb (`:8080`) — logged in the UI.
2. → CLIProxyAPI (`:8317`) — swap to OAuth, forward upstream.
3. → Anthropic — response returns the same path in reverse.

---

## 4. What to look at in mitmweb

This is the part of the exercise that actually matters. Open <http://localhost:8082>, run a few normal `claude` commands, and read each flow:

- **First request of the session** — note the `system` prompt Claude Code injects on its own (you didn't type it), the `tools` array (every MCP server, sub-agent, slash command shows up as a tool definition), and the `model`.
- **Each subsequent turn** — the entire `messages` array of all prior turns is replayed verbatim. This is the point the README hammers on: **stateless every turn**. Watching the payload grow turn by turn convinces you faster than any explanation.
- **After `/compact`** — call `/compact`, then send one more message. Compare the `messages` array before and after: most prior turns get replaced by a single summary block. This is the mechanic explained in [Section 2.3 of the README](../README.md#23-delegate-dont-dictate-silently-breaks-in-long-sessions).
- **When Claude calls a tool** — the request gains a `tool_use` block, the next response carries a `tool_result`. The full lifecycle fits in a few consecutive flows.
- **Streaming SSE** — Claude Code uses `stream: true`. The UI shows the chunks as they arrive; you literally watch the text generated piece by piece.

Filter to only show requests that hit the Anthropic endpoint (type into the search bar at the top of the UI):

```
~u /v1/messages
```

Small tip: drag the `Size` column wider to see how the body inflates per turn — the most concrete picture of the "context cost" the README spends an entire section on.

---

## 5. Cleanup

```bash
# Stop mitm: Ctrl-C in terminal 2
# Stop cliproxy: Ctrl-C in terminal 1, or:
docker stop cliproxy

# Drop env:
unset ANTHROPIC_BASE_URL ANTHROPIC_API_KEY

# Wipe stored login (only if you want to re-login):
rm -rf ~/cliproxy/auths/*
```

---

## Troubleshooting

| Symptom | Common cause |
| --- | --- |
| `connection refused` from Claude | mitm not started, or cliproxy not on `:8317`. Test: `curl localhost:8317/v1/models`. |
| `401 Unauthorized` | `ANTHROPIC_API_KEY` does not match `api-keys` in `config.yaml`. |
| `Provider claude is in cooldown` | Claude token rate-limited or revoked — re-run `--claude-login`. |
| Login command hangs, no URL | Port `54545` not exposed. Or running headless → add `-no-browser`. |
| Port `8081` `address already in use` | Remap the UI port: `-p 8082:8081` (already done in this guide). |
| mitm UI keeps prompting for password | Restart with `--set web_password=admin`. |
| Body fails to parse as JSON | Check **Request → Raw** in the mitm UI: first byte must be `{` and `Content-Type: application/json`. |
| Claude still hits `api.anthropic.com` | `ANTHROPIC_BASE_URL` not exported in the terminal running `claude`. Verify with `echo $ANTHROPIC_BASE_URL`. |

---

## Notes

- Containers reach the host via `host.docker.internal` on macOS/Windows. On bare Linux, swap it for the host IP or use `--network host`.
- Tokens live in `~/cliproxy/auths/`; back them up if you migrate machines, otherwise you'll need to re-login.
- mitm's reverse mode only forwards the configured HTTP/HTTPS target; you do **not** need to install a CA cert in the client (unlike explicit-proxy mode).
- This pattern isn't tied to Claude — point the reverse target at Codex, Gemini, or a self-hosted simulator and the exercise is the same. The lesson generalizes: the client always talks through a proxy you control, and what you control you can see.

## Sources

- [router-for-me/CLIProxyAPI (GitHub)](https://github.com/router-for-me/CLIProxyAPI)
- [Reference docker-compose.yml for CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI/blob/main/docker-compose.yml)
- [mitmproxy reverse mode docs](https://docs.mitmproxy.org/stable/concepts-modes/#reverse-proxy)
- [Anthropic API: ANTHROPIC_BASE_URL env](https://docs.anthropic.com/en/api/getting-started)
