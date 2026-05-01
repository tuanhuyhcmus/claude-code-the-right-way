# POC: pair Claude Code với một model "chưa xịn" qua Ollama

> Companion cho [Phần 1 của README](../README.vi.md) — đoạn *"Cứ thử pair một client xịn với 1 model chưa xịn xem (vd các model < 30B param) bạn sẽ thấy dù context có tốt đến đâu thì nó vẫn không hoạt động theo cách làm bạn hài lòng."*
>
> Đây là POC chị em với [client → proxy → server](./poc-client-proxy-server.vi.md). Cái trước cho bạn nhìn **thứ Claude Code gửi đi**; cái này cho bạn cảm nhận **thứ Claude Code đáng lẽ phải nhận về**. Khi tự tay chạy, bạn sẽ hiểu vì sao README cứ nhấn vào model — và vì sao "đổ thêm context" không cứu được nếu phía server yếu.

Setup gồm hai phần:

- **Ollama** — chạy local, bind ở `:11434`. Từ phiên bản `0.14` (đầu 2026), Ollama đã có sẵn **Anthropic Messages API compat layer** ở chính endpoint đó, nên **không cần proxy translation** nữa.
- **Claude Code CLI** — trỏ `ANTHROPIC_BASE_URL` về Ollama, chọn model đã pull về máy.

```
Claude Code ──► localhost:11434 (Ollama, Anthropic-compatible) ──► gemma3:4b (local)
```

So với POC kia (CLIProxyAPI + mitm), setup này **đơn giản hơn nhiều** — không có proxy, không có OAuth — vì mục tiêu là cảm nhận "model yếu là cảm giác gì", không phải bắt request.

---

## 1. Cài Ollama

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
ollama --version       # phải >= 0.14 để có Anthropic-compat layer
```

Khởi chạy daemon (giữ chạy ở terminal riêng hoặc dùng `brew services start ollama`):

```bash
ollama serve
```

> Mặc định Ollama bind `127.0.0.1:11434`. Nếu cần expose ra LAN: `OLLAMA_HOST=0.0.0.0:11434 ollama serve`.

## 2. Pull một model "chưa xịn"

Ý đồ POC là chọn model **vừa đủ chạy được tool calling, vừa đủ yếu để bạn thấy giới hạn**. Bảng các tag Gemma 3 hiện có:

| Tag           | Size  | Context |
| ------------- | ----- | ------- |
| `gemma3:270m` | 292MB | 32K     |
| `gemma3:1b`   | 815MB | 32K     |
| `gemma3:4b`   | 3.3GB | 128K    |
| `gemma3:12b`  | 8.1GB | 128K    |
| `gemma3:27b`  | 17GB  | 128K    |

Khuyến nghị cho POC:

- **`gemma3:4b`** — sweet spot cho mục tiêu "thấy giới hạn". 128K context (đủ cho session Claude Code thật), kích thước vừa, chạy được trên 16GB RAM. Đây là model README ám chỉ khi nói "< 30B param".
- `gemma3:12b` nếu máy bạn khoẻ — vẫn thua xa Sonnet/Opus, đủ để cảm thấy chênh lệch.
- **Đừng** chọn `gemma3:1b` hay `:270m` cho POC này — context 32K sẽ chết ngay khi Claude Code nhồi system prompt + tools array vào turn đầu tiên, bạn sẽ chỉ thấy lỗi context-overflow chứ không cảm nhận được cái cần cảm nhận.

```bash
ollama pull gemma3:4b
```

Smoke-test (qua endpoint Ollama gốc):

```bash
ollama run gemma3:4b "viết hàm fibonacci bằng Go"
```

Nếu in ra code — ổn.

## 3. Trỏ Claude Code về Ollama

Hai cách, kết quả như nhau.

### 3.1. Cách đơn giản — dùng `ollama launch`

Ollama 0.14+ có wrapper sẵn để export env và launch claude:

```bash
ollama launch claude --model gemma3:4b
```

### 3.2. Cách "manual" — tự export env

```bash
export ANTHROPIC_BASE_URL=http://localhost:11434
export ANTHROPIC_AUTH_TOKEN=ollama
export ANTHROPIC_API_KEY=""

claude --model gemma3:4b
```

> Một số phiên bản `claude` CLI bỏ qua `ANTHROPIC_API_KEY` rỗng và đọc từ `ANTHROPIC_AUTH_TOKEN`. Nếu gặp lỗi `Missing API key`, set thêm `ANTHROPIC_API_KEY=ollama`.

## 4. Cảm nhận gì khi dùng

Đây là phần *thật sự đáng* của POC. Mở session Claude Code mới và thử lần lượt vài tác vụ quen thuộc, **so sánh trực tiếp** với Sonnet/Opus đã quen:

- **Latency turn đầu** — Gemma3:4b trên CPU/Apple Silicon thường mất vài giây cho turn đầu. Sonnet trả lời gần như tức thì. Đây là cảm giác đầu tiên bạn nhận ra: tốc độ là một dạng UX, không chỉ là số trên benchmark.
- **Tool calling** — yêu cầu Claude `đọc file README.md và tóm tắt`. Quan sát:
  - Model có gọi đúng tool `Read`?
  - Có gọi sai tên tool, hay chế ra tool không tồn tại không?
  - Có tóm tắt sai/bịa nội dung không?
- **Multi-step plan** — yêu cầu một việc phải gọi 3-4 tool liên tiếp (vd: `tìm tất cả file .go có TODO, đọc 3 file đầu, đề xuất fix`). Theo dõi xem nó dừng giữa chừng, lặp lại, hay đi sai hướng từ bước 2.
- **Long context** — paste 5 file source vào prompt rồi hỏi một câu cần tổng hợp. Đây là chỗ mà Sonnet/Opus toả sáng và Gemma 4B ngã rõ nhất — không phải vì context window bé (Gemma3:4b có 128K) mà vì **self-attention dilution** ([deep dive ở `docs/llm-mechanics/self-attention.vi.md`](./llm-mechanics/self-attention.vi.md)).
- **Behaviour với CLAUDE.md/skills** — nếu repo có sẵn `CLAUDE.md` và rule, model yếu thường ignore hoặc hiểu sai. Đây là minh hoạ trực tiếp cho luận điểm xuyên suốt README: **client + context tốt vô nghĩa nếu model không đủ năng lực reasoning**.

Nếu muốn quan sát chi tiết hơn, **kết hợp với POC kia**: chạy mitmweb reverse-proxy đứng giữa Claude Code và Ollama:

```bash
docker run --rm -it -p 8080:8080 -p 8082:8081 \
  mitmproxy/mitmproxy mitmweb \
    --mode reverse:http://host.docker.internal:11434 \
    --web-host 0.0.0.0 \
    --set web_open_browser=false \
    --set web_password=admin

export ANTHROPIC_BASE_URL=http://localhost:8080   # qua mitm thay vì :11434
claude --model gemma3:4b
```

Bạn sẽ thấy cùng một system prompt + tools array mà Sonnet xử lý mượt thì Gemma3:4b trả về tool call lệch — minh chứng trực quan nhất cho luận điểm "client xịn không cứu nổi model yếu".

## 5. Cleanup

```bash
unset ANTHROPIC_BASE_URL ANTHROPIC_AUTH_TOKEN ANTHROPIC_API_KEY

# Stop Ollama
brew services stop ollama       # macOS
# hoặc Ctrl-C terminal đang chạy `ollama serve`

# Xoá model nếu cần dọn ổ
ollama rm gemma3:4b
```

---

## Troubleshooting

| Triệu chứng                                     | Nguyên nhân thường gặp                                                                                                          |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `connection refused` từ Claude                  | Ollama chưa `serve`. Verify: `curl http://localhost:11434/api/tags`.                                                            |
| `model not found`                               | Chưa `ollama pull <tag>`. Liệt kê model sẵn có: `ollama list`.                                                                  |
| `404` hoặc lỗi format Anthropic                 | Ollama < 0.14 chưa có Anthropic-compat. Update: `brew upgrade ollama` hoặc cài lại từ `install.sh`.                             |
| Claude bị treo / timeout                        | Model quá lớn so với RAM máy → Ollama swap. Đổi sang tag nhỏ hơn (`gemma3:4b` thay `:12b`).                                     |
| Tool calling sai liên tục                       | Đặc trưng của model nhỏ. Đó *chính là* điều POC này muốn bạn thấy — không phải bug để fix.                                     |
| `context length exceeded`                       | Bạn chọn `gemma3:1b`/`:270m` (32K). Đổi sang `:4b` (128K).                                                                      |
| `Missing API key`                               | Set thêm `export ANTHROPIC_API_KEY=ollama` (vài bản `claude` không chấp nhận empty).                                            |
| Reply tốt một cách bất ngờ                      | Bạn vô tình gọi qua `kimi-k2.5:cloud` / `glm-5:cloud` (Ollama Cloud, không phải local). Check `ollama list` để chắc đang local. |

---

## Ghi chú

- Pattern này không gắn riêng với Gemma. Đổi sang `qwen3:4b`, `llama3.2:3b`, `phi3:3.8b`... cũng được — kết luận giống nhau: **tool-using agent + small local model = trải nghiệm rõ rệt kém hơn cùng client + frontier model**.
- "Local" ≠ "free": tốn RAM, tốn pin, tốn thời gian. Nếu mục tiêu là *làm việc thật*, kết luận thường là quay về Sonnet/Opus. Nếu mục tiêu là *học*, đây là một trong những cách rẻ nhất để cảm nhận giới hạn.
- Đừng dùng kết quả POC này để đánh giá Gemma 3 — Gemma 3 chạy task khác (chat, summarisation, classification) có thể rất tốt. Bài này chỉ chứng minh nó chưa đủ cho **tool-using long-horizon agent** mà Claude Code thiết kế cho.

## Sources

- [Ollama: Claude Code integration docs](https://docs.ollama.com/integrations/claude-code)
- [Ollama Blog: Claude Code with Anthropic API compatibility](https://ollama.com/blog/claude)
- [Ollama Library: Gemma 3](https://ollama.com/library/gemma3)
- [Anthropic API: ANTHROPIC_BASE_URL env](https://docs.anthropic.com/en/api/getting-started)
