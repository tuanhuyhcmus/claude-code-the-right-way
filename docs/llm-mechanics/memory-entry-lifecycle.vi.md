# Vòng đời của từng mục trong `/memory`

*[English](./memory-entry-lifecycle.md) · **Tiếng Việt***

*Cái gì được load vào lúc nào, cái gì sống sót qua `/compact`, và prompt cache của Anthropic thực sự lưu gì.*

`/memory` cho bạn một snapshot mọi thứ đang được load từ disk vào context của session: mọi `CLAUDE.md`, mọi rule đã load, mọi auto-memory file. Điều nó không nói cho bạn là **các mục trong snapshot đó không sống cùng một kiểu**. Biết mỗi mục load ra sao, sống sót qua compaction thế nào, và vào prompt cache lúc nào — đó là thứ phân biệt giữa *"thứ Claude đọc một lần"* với *"thứ Claude tiếp tục trả phí ở mỗi turn."*

Tài liệu này là bảng tham chiếu cho chuyện đó. Nó cố tình ngắn và load-bearing — README chính link đến đây mỗi khi cần chi tiết.

---

## Bảng

| Cái bạn thấy trong `/memory`                                        | Khi nào nó vào context                                  | Chuyện gì xảy ra sau `/compact`                                                                              | Có nằm trong prompt cache không?                                      |
| :------------------------------------------------------------------ | :------------------------------------------------------ | :----------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------- |
| Project / user `CLAUDE.md`                                          | Một lần ở đầu session                                   | Re-inject từ disk                                                                                            | Có — nằm trong prefix cache 5 phút (tùy chọn 1 giờ)                   |
| Always-loaded rules (`.claude/rules/*.md` không có `paths:`)        | Một lần ở đầu session                                   | Re-inject từ disk                                                                                            | Cùng prefix với `CLAUDE.md`                                           |
| Auto memory (`MEMORY.md`)                                           | Một lần ở đầu session (200 dòng đầu / 25KB)             | Re-inject từ disk                                                                                            | Cùng prefix                                                           |
| Skill **descriptions** (chỉ metadata)                               | Một lần ở đầu session                                   | **Mất.** Danh sách skill là khối duy nhất không được re-inject — chỉ những skill bạn đã invoke mới sống sót  | Cùng prefix (trong khi còn ở trong context)                           |
| Path-scoped rules (`paths: src/api/**`)                             | Khi Claude đọc một file khớp, giữa session              | Mất cho đến khi một file khớp được đọc lại                                                                   | Chỉ vào cache sau lần inject đầu                                      |
| Nested `CLAUDE.md` trong subdirectory                               | Khi Claude đọc một file trong subdirectory đó           | Mất cho đến khi một file trong subdirectory đó được đọc lại                                                  | Giống path-scoped rules                                               |
| Memory topic files (mọi thứ trong `memory/` trừ `MEMORY.md`)        | Chỉ khi Claude đọc một cái một cách tường minh          | Chỉ sống sót nếu đã được đọc                                                                                 | Chỉ vào cache sau lần inject đầu                                      |
| Skill **bodies**                                                    | Khi được invoke (bởi bạn qua `/name` hoặc bởi Claude)   | Re-inject, giới hạn 5,000 token mỗi skill và 25,000 token tổng; cái cũ nhất bị bỏ trước                      | Chỉ vào cache sau lần inject đầu                                      |
| Hooks                                                               | N/A — hook chạy trong harness, không phải trong context | N/A                                                                                                          | Không bao giờ ở trong context, không bao giờ được cache                |

---

## Hai hệ quả không hiển nhiên

### 1. "Loaded at session start" không phải một nhóm duy nhất

`CLAUDE.md`, always-loaded rules, auto-memory, và skill description đều xuất hiện trước prompt đầu tiên của bạn. Dễ nghĩ rằng chúng là một nhóm. Không phải. Skill description là thứ duy nhất `/compact` âm thầm bỏ. Hệ quả có một bất đối xứng tinh tế trở nên quan trọng ngay lúc bạn bắt đầu dựa vào skill:

- **`/command` tường minh vẫn chạy sau `/compact`.** Gõ `/skill-name` đi qua harness Claude Code (client), nó load skill body trực tiếp từ disk và inject vào cuộc hội thoại. Claude không cần biết skill trong context để route một `/command` — harness nắm menu.
- **Auto-trigger implicit bị hỏng**, với bất kỳ skill nào Claude chưa từng invoke trong session này. Claude không còn thấy danh sách description, nên không biết những skill đó tồn tại. Mọi skill có `disable-model-invocation: true` không bị ảnh hưởng (description của nó không bao giờ nằm trong context ngay từ đầu); mọi thứ khác phụ thuộc việc Claude auto-route đều âm thầm ngừng fire.
- **Kết quả ròng:** framework auto-triggering "magic" của bạn rõ ràng đần đi sau một session dài — không phải vì model tệ đi, mà vì một nửa bản đồ routing của nó đã bị tóm tắt đi mất. Đây là một trong những lời than phổ biến nhất *"Claude trước đây làm X tự động, giờ không nữa"*, và luôn có cùng một nguyên nhân.

### 2. Path-scoped rule và nested `CLAUDE.md` là những mục dễ gãy nhất

Cả hai đều sống trong message history thay vì prefix. Compaction xóa chúng, và chúng chỉ quay lại khi Claude tình cờ đọc lại một file khớp. Nếu một rule phải tồn tại qua compaction, hãy bỏ scope `paths:` hoặc hoist nội dung lên `CLAUDE.md` ở project root. Còn không, chấp nhận rằng sự hiện diện của rule là có điều kiện — phụ thuộc vào những file Claude đã đọc gần đây.

---

## Nguồn

Đáng đọc trực tiếp — đặc biệt cái đầu tiên, nó minh họa rất nhiều thứ ở đây một cách trực quan:

- [Claude Code · Explore the context window](https://code.claude.com/docs/en/context-window) — timeline tương tác của một session mô phỏng, kèm bảng *"What survives compaction"*.
- [Claude Code · Memory](https://code.claude.com/docs/en/memory) — hệ thống CLAUDE.md, rule, và auto-memory.
- [Anthropic · Prompt caching](https://platform.claude.com/docs/en/docs/build-with-claude/prompt-caching) — TTL, giá, và cái gì được cache.

---

## Deep dive liên quan

- [`why-context-matters.md`](./why-context-matters.md) — ba fact về LLM runtime khiến lifecycle này load-bearing ngay từ đầu.
- [`per-turn-cost-math.md`](./per-turn-cost-math.md) — cache hit, cache write, và mid-session invalidation thực sự tốn bao nhiêu mỗi turn.
