# Xây dashboard còn thiếu — lời kêu gọi người đóng góp

*[English](./dashboard-rfc.md) · **Tiếng Việt***

*RFC cho một công cụ inspect context sống. Tách khỏi README chính vì đây là nhánh phụ của luận điểm chính — nhưng là một nhánh có thể bắt tay vào làm.*

Thesis trung tâm của README chính là: **hiểu Claude đang biết gì ngay lúc này là một kỹ năng, và công cụ để nhìn thấy điều đó đang bị đầu tư thiếu.** Câu lệnh built-in `/context` là view chẩn đoán tốt nhất hiện có, và nó là một đống text. Bản ["Explore the context window"](https://code.claude.com/docs/en/context-window#what-the-timeline-shows) tương tác của Anthropic là một simulation hardcode trong docs của họ — không phải inspector sống cho session của chính bạn. Các nỗ lực open-source gần nhất đều hụt theo những cách cụ thể, giải quyết được: [`claude-devtools`](https://github.com/matt1398/claude-devtools) gộp các skill vào một bucket duy nhất; [`token-dashboard`](https://github.com/nateherkai/token-dashboard) chỉ track skill đã invoke, không track description; [`cc-dump`](https://github.com/brandon-fryslie/cc-dump) hiển thị system prompt thô nhưng không parse.

Khoảng trống là có thật, và — sau khi tự kiểm chứng bằng việc đọc session JSONL của chính mình trong lúc viết repo này — nó **xây được**. Ba fact chúng tôi đã xác nhận:

1. **Session JSONL chứa khoảng 80% dữ liệu.** Mọi file `~/.claude/projects/<project>/<session-id>.jsonl` chứa các attachment có cấu trúc (`skill_listing`, `nested_memory`, `deferred_tools_delta`, `hook_success`, và ~30 loại khác) với text thật của mọi skill description, rule, `CLAUDE.md` nested, và tên MCP tool đã vào context. Mỗi assistant message ghi lại chính xác `cache_creation_input_tokens`, `cache_read_input_tokens` mỗi turn, và các breakdown `ephemeral_5m` / `ephemeral_1h`.
2. **Phần còn thiếu nhỏ và giải quyết được.** Text của system prompt không nằm trong JSONL, nhưng nó có ở [`Piebald-AI/claude-code-system-prompts`](https://github.com/Piebald-AI/claude-code-system-prompts), cập nhật theo từng version Claude Code. Schema đầy đủ của MCP tool cũng không nằm trong JSONL, nhưng `/mcp` và MCP server có thể cung cấp. Attribution theo token là một call đến endpoint [`count_tokens`](https://platform.claude.com/docs/en/api/messages-count-tokens) miễn phí của Anthropic.
3. **Chưa có tool nào ghép những thứ này lại ở độ chi tiết của `/context`**, chứ chưa nói đến theo thời gian, xuyên team, hay kèm visualize cache-lifecycle.

## Một build plan từng bước

| Stage | Phạm vi | Effort | Dành cho ai |
| :---- | :------ | :----- | :----------- |
| **MVP** — CLI snapshot | Đọc một file JSONL, xuất breakdown token tĩnh theo skill / theo rule / theo tool khớp với output `/context`. Tokenize qua `count_tokens`, cache local. | ~1 tuần, solo | Kỹ sư cá nhân muốn audit setup của chính mình |
| **V2** — live TUI | Watch file JSONL đang được ghi trong một session đang chạy (`fswatch` / `inotify`), cập nhật một UI terminal với category bar + panel chi tiết. | +2–3 tuần | Người dùng Claude Code hằng ngày, power user |
| **V3** — web dashboard | Backend serve parsed state qua WebSocket; timeline React/Svelte có drilldown lấy cảm hứng từ simulation của Anthropic; parse sidechain subagent; so sánh trước/sau `/compact`; timeline cache hit/miss. | +1–2 tháng, 2 contributor | Team, demo PR, conference talk |
| **V4** — team analytics | Tổng hợp session đa người dùng, alert chi phí, phát hiện bất thường, chế độ bảo vệ quyền riêng tư strip nội dung hội thoại. | +1–3 tháng, 3–4 contributor | Leadership kỹ thuật, finance, team platform |

Mỗi stage tự thân đều hữu ích. Riêng MVP đã ăn đứt mọi giải pháp open-source hiện có.

## Câu hỏi design còn mở

- **License.** MIT, Apache, hay source-available kèm hạn chế thương mại?
- **Phạm vi.** Parser có nên xử lý cả các agentic CLI khác có layout JSONL tương tự (OpenCode, Cline, Codex, OpenClaw), hay giữ tập trung vào Claude Code?
- **Monetization.** Open-source core kèm tier team trả phí, hay full OSS và dựa vào sponsorship?
- **Hosting.** Chỉ self-hosted, hay có option SaaS hosted cho team dashboard?
- **Ổn định schema.** Format JSONL của Claude Code có thể đổi không báo trước. Pin theo ma trận version và integration-test, hay parse phòng thủ và warn khi gặp cái chưa biết?

## Cách tham gia

- Mở issue trên repo chính kèm quan điểm của bạn về phạm vi hoặc một tier bạn muốn nhận.
- Nếu bạn đã xây cái gì đó lân cận ([`ccusage`](https://github.com/ryoppippi/ccusage), [`token-dashboard`](https://github.com/nateherkai/token-dashboard), [`claude-devtools`](https://github.com/matt1398/claude-devtools), [`cc-dump`](https://github.com/brandon-fryslie/cc-dump), [`proxyclawd`](https://github.com/dyshay/proxyclawd)) và muốn hợp tác thay vì cạnh tranh, nói một tiếng — khoảng trống nhiều hơn số người đang cố lấp.

Con đường nhanh nhất có lẽ là: một contributor lo core parser JSONL, một lo tokenization và tính chi phí, một lo UI. Ba người, ba tháng, một tool chưa ai từng xây.
