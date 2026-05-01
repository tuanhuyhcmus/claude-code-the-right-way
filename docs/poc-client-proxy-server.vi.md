# POC: client → proxy → server — bắt request Claude Code thực sự gửi đi

> Companion cho [Phần 1 của README](../README.vi.md) — đoạn *"tôi muốn bạn tự setup một môi trường POC: client (Claude Code) → proxy (cái nào cũng được) → server"*.
>
> Mục tiêu của buổi POC này không phải là cài thêm tool, mà là **nhìn tận mắt** cái Claude Code thật sự gửi đi: system prompt nó tự nhồi vào, mảng `tools` nó khai báo, lịch sử messages nó gửi lại mỗi turn, độ dài thật của một context window mà bạn cảm giác là "vài câu chat". Hầu hết trực giác về context và Claude Code trong README sẽ tự đến sau khi bạn nhìn mấy flow đầu tiên.

Setup gồm ba thành phần, mỗi cái chạy trong một container riêng:

- **CLIProxyAPI** (`:8317`) — đứng vai trò **server**. Nó wrap OAuth Claude Code/Pro/Max thành một endpoint Anthropic-compatible (`POST /v1/messages`). Nhờ đó Claude Code "tưởng" mình đang gọi `api.anthropic.com`.
- **mitmweb** (`:8080` cho traffic, `:8082` cho UI) — đứng vai trò **proxy**. Reverse-mode: client gọi vào `:8080`, mitm log lại rồi forward sang `:8317`.
- **Claude Code CLI** — đứng vai trò **client**. Trỏ `ANTHROPIC_BASE_URL` về mitm để mọi request đi qua đó.

```
Claude Code ──► localhost:8080 (mitmweb)  ──►  host.docker.internal:8317 (CLIProxyAPI)  ──►  api.anthropic.com (OAuth)
                       │
                       └──► UI: http://localhost:8082
```

> Tại sao cần CLIProxyAPI mà không trỏ thẳng Claude Code → mitm → `api.anthropic.com`? Vì Claude Code dùng OAuth (Pro/Max) cho header auth, không phải `x-api-key` đơn giản. CLIProxyAPI giữ token OAuth và đổi về header phù hợp khi gọi upstream — bạn không phải đụng tới cơ chế auth, chỉ tập trung quan sát body request. Nếu bạn dùng API key trả phí thuần (không qua subscription), có thể bỏ CLIProxyAPI, đặt mitm reverse `api.anthropic.com` luôn.

---

## 1. Khởi động CLIProxyAPI (server)

> Repo: <https://github.com/router-for-me/CLIProxyAPI> — image dùng sẵn: `eceasy/cli-proxy-api:latest`.

### 1.1. Chuẩn bị thư mục & config

```bash
mkdir -p ~/cliproxy/auths
cd ~/cliproxy
```

Tạo `config.yaml` tối thiểu:

```yaml
host: ""           # bind tất cả interface; đổi "127.0.0.1" để chỉ cho local
port: 8317

auth-dir: "/root/.cli-proxy-api"

# API key client gọi vào (Claude Code sẽ dùng giá trị này ở ANTHROPIC_API_KEY).
# Đặt giá trị bất kỳ — vì đứng sau mitm, không expose ra ngoài.
api-keys:
  - "proxy"

debug: true
logging-to-file: false

# Round-robin nếu sau này login thêm account
routing:
  strategy: "round-robin"
  session-affinity: false
```

### 1.2. Login Claude (one-time, OAuth)

Login mở browser ở host, callback về port `54545`. Vì chạy trong Docker nên expose port này:

```bash
docker run --rm -it \
  -p 54545:54545 \
  -v "$PWD/config.yaml:/CLIProxyAPI/config.yaml" \
  -v "$PWD/auths:/root/.cli-proxy-api" \
  eceasy/cli-proxy-api:latest \
  /CLIProxyAPI/CLIProxyAPI --claude-login
```

- Container in một URL `https://claude.ai/...` → mở browser host, đăng nhập tài khoản Claude (Pro/Max).
- Sau khi xong, callback về `http://localhost:54545/...`, container ghi token vào `~/cliproxy/auths/`, rồi exit.
- Verify: `ls ~/cliproxy/auths/` thấy file JSON token.

> Máy headless (không browser): thêm cờ `-no-browser` để CLI in URL ra terminal, mở ở máy khác.

### 1.3. Chạy CLIProxyAPI ở chế độ server

Terminal 1 (giữ chạy):

```bash
docker run --rm -it \
  --name cliproxy \
  -p 8317:8317 \
  -v "$PWD/config.yaml:/CLIProxyAPI/config.yaml" \
  -v "$PWD/auths:/root/.cli-proxy-api" \
  eceasy/cli-proxy-api:latest
```

Smoke-test trực tiếp (chưa qua mitm):

```bash
curl -s http://localhost:8317/v1/models -H "x-api-key: proxy" | jq .
```

Nếu thấy danh sách model → server OK.

### 1.4. (Tùy chọn) Login thêm provider khác

CLIProxyAPI cũng bọc được Codex / Gemini, hữu ích nếu bạn muốn so sánh client OpenAI Codex hoặc Gemini CLI cùng pattern proxy:

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

## 2. Khởi động mitmweb (proxy)

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

- `8080` → port client (Claude Code) gọi vào.
- `8082` → web UI mitmweb (mapping nội bộ `8081`).
- `--mode reverse:...` → forward sang CLIProxyAPI ở host.
- `--set web_password=admin` → password đăng nhập UI.

Mở UI: <http://localhost:8082> (password `admin`).

---

## 3. Chạy Claude Code qua proxy (client)

Terminal 3:

```bash
export ANTHROPIC_BASE_URL=http://localhost:8080
export ANTHROPIC_API_KEY="proxy"        # khớp với api-keys trong config.yaml
claude --model opus-4-6
```

Mọi request `POST /v1/messages` từ Claude Code giờ đi qua chuỗi:

1. → mitmweb (`:8080`) — log lại trong UI.
2. → CLIProxyAPI (`:8317`) — đổi sang OAuth, forward upstream.
3. → Anthropic — response trả về theo đường ngược.

---

## 4. Đọc gì trong mitmweb

Đây là phần *thật sự đáng giá* của buổi POC. Mở <http://localhost:8082>, gõ `claude` vài lệnh quen thuộc và xem từng flow:

- **Request đầu tiên của session** — chú ý `system` prompt mà Claude Code tự nhồi vào (không phải bạn gõ), mảng `tools` (mọi MCP server, sub-agent, slash command đều biến thành tool definition), và `model`.
- **Mỗi turn tiếp theo** — toàn bộ `messages` của các turn cũ được gửi lại nguyên văn. Đây là điều README nhắc đi nhắc lại: **stateless mỗi turn**. Nhìn payload tăng dần, bạn sẽ tin nó hơn là đọc.
- **Sau `/compact`** — gọi `/compact` rồi gửi 1 message tiếp theo. So sánh `messages` array trước và sau: phần lớn turn cũ bị thay bằng một block tóm tắt. Đây là cơ chế ở [Mục 2.3 của README](../README.vi.md#23-delegate-dont-dictate-âm-thầm-đổ-vỡ-trong-session-dài).
- **Khi Claude gọi tool** — request có thêm khối `tool_use`, response sau đó có `tool_result`. Lifecycle hoàn chỉnh nằm gọn trong vài flow liên tiếp.
- **Streaming SSE** — Claude Code dùng `stream: true`. UI hiển thị từng chunk; bạn thấy text được generate dần dần đúng theo thứ tự client nhận.

Filter chỉ hiện request đến endpoint Anthropic (gõ vào ô search trên cùng UI):

```
~u /v1/messages
```

Một mẹo nhỏ: kéo cột `Size` để thấy độ phình của body theo turn — số liệu trực quan nhất cho khái niệm "context cost" mà README dành nguyên một mục để bàn.

---

## 5. Cleanup

```bash
# Dừng mitm: Ctrl-C ở terminal 2
# Dừng cliproxy: Ctrl-C ở terminal 1, hoặc:
docker stop cliproxy

# Bỏ env:
unset ANTHROPIC_BASE_URL ANTHROPIC_API_KEY

# Xoá token đã login (nếu muốn re-login):
rm -rf ~/cliproxy/auths/*
```

---

## Troubleshooting

| Triệu chứng | Nguyên nhân thường gặp |
| --- | --- |
| `connection refused` từ Claude | mitm chưa start, hoặc cliproxy không chạy ở `:8317`. Test: `curl localhost:8317/v1/models`. |
| `401 Unauthorized` | `ANTHROPIC_API_KEY` không khớp `api-keys` trong `config.yaml`. |
| `Provider claude is in cooldown` | Token Claude bị rate-limit hoặc revoke — login lại bằng `--claude-login`. |
| Login command treo, không in URL | Chưa expose port `54545`. Hoặc chạy headless → thêm cờ `-no-browser`. |
| Port `8081` báo `address already in use` | Đổi mapping web UI: `-p 8082:8081` (đã làm trong hướng dẫn). |
| UI mitm bắt nhập password mãi | Restart với `--set web_password=admin`. |
| Body JSON parse fail | Xem **Request → Raw** trong mitm UI: ký tự đầu tiên phải là `{` và `Content-Type: application/json`. |
| Claude vẫn gọi `api.anthropic.com` | `ANTHROPIC_BASE_URL` chưa export trong terminal đang chạy `claude`. Verify `echo $ANTHROPIC_BASE_URL`. |

---

## Ghi chú

- Container truy cập host bằng `host.docker.internal` (macOS/Windows). Linux thuần: thay bằng IP host hoặc `--network host`.
- Token nằm ở `~/cliproxy/auths/`; backup nếu chuyển máy để khỏi login lại.
- Reverse-mode mitm chỉ forward HTTP/HTTPS đã cấu hình; **không** cần cài CA cert vào client (khác với explicit-proxy mode).
- Pattern này không gắn riêng với Claude — bạn có thể đổi target sang Codex/Gemini/llm-d-inference-sim cùng cách. Ý nghĩa POC giữ nguyên: client luôn nói chuyện qua một proxy bạn kiểm soát, bạn kiểm soát được tức bạn nhìn được.

## Sources

- [router-for-me/CLIProxyAPI (GitHub)](https://github.com/router-for-me/CLIProxyAPI)
- [docker-compose.yml mẫu của CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI/blob/main/docker-compose.yml)
- [mitmproxy reverse mode docs](https://docs.mitmproxy.org/stable/concepts-modes/#reverse-proxy)
- [Anthropic API: ANTHROPIC_BASE_URL env](https://docs.anthropic.com/en/api/getting-started)
