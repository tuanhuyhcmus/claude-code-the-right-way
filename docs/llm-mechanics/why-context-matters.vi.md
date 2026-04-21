# Vì sao context quan trọng: ba fact về LLM runtime

*[English](./why-context-matters.md) · **Tiếng Việt***

*Những lý do cơ học khiến mỗi primitive của Claude Code tồn tại.*

Doc này là nền tảng cho phần lập luận của README chính. Mọi quyết định placement trong repo này — cái gì vào `CLAUDE.md`, cái gì vào rule, cái gì vào skill, cái gì ở ngoài model trong hook — đều chảy ra từ ba fact về cách LLM thực sự chạy tại inference time. Nếu bạn đã rành cách LLM hoạt động ở runtime, có thể bỏ qua. Nếu chưa, bắt đầu ở đây trước khi đọc bất kỳ thứ gì khác.

---

## Fact 1 — Context window hữu hạn, và được dùng chung

Model chỉ có thể "nhìn" một số token cố định cùng một lúc — đó là context window của nó. Claude Opus 4.7 trần ở 1M token. Nghe khổng lồ. Thật ra không, vì budget được **dùng chung**, không dành riêng cho câu hỏi của bạn.

Mọi thứ bên dưới tranh cùng 1M token đó ở mỗi turn:

- System prompt của chính Claude Code (vài nghìn token chỉ để định nghĩa hành vi của nó)
- JSON schema của mọi tool — built-in tool, MCP tool, agent definition (dễ dàng 10–20k token cho một setup điển hình)
- Mọi `CLAUDE.md` được load bằng directory walk-up
- Mọi `.claude/rules/*.md` luôn-load
- Description của mọi skill (không phải body — chỉ metadata — nhưng cho *mọi* skill)
- `MEMORY.md` (tới 25KB)
- Toàn bộ hội thoại trước: message của bạn, reply của Claude, và mọi tool result
- Nội dung file Claude đã đọc, output của grep, output của bash
- Task output Claude đang build ngay lúc này

Một lần `Read` file source 2000 dòng có thể ăn 30–50k token. Mười lần đọc như vậy trong một session debug dài và bạn đã đốt 5% window trước khi làm bất kỳ việc thật nào. Bảo Claude dump một file 500 dòng ba lần — 150k token biến mất, cạnh tranh với skill description mà bạn cần Claude trigger.

**"1M token" là trần, không phải workspace.** Workspace dùng được co lại mỗi phút trong session.

---

## Fact 2 — Mỗi request gửi lại toàn bộ hội thoại

Inference LLM là **stateless**. Model không nhớ gì giữa các request. Mỗi lần Claude trả lời, client gửi lại toàn bộ hội thoại trước, mọi `CLAUDE.md` đã load, mọi rule, mọi skill description, mọi tool definition — tất cả, từ turn 1.

Hệ quả thực tế: nếu bạn thêm 500 dòng instruction vào `CLAUDE.md`, bạn trả tiền cho 500 dòng đó trên mỗi response trong suốt phần còn lại của session. "Khoản thuế trả trước" không phải chi phí một lần — nó là chi phí mỗi turn nhân với mọi turn bạn từng có.

Đây là chỗ prompt caching cứu bạn. Anthropic cache prefix ổn định của mỗi request ở server trong 5 phút (hoặc 1 giờ với chi phí ghi 2×), nên lần gửi lại ở turn N+1 chỉ trả 0.1× cho phần prefix khớp với turn N. Nhưng cache là prefix-matched: **bất kỳ thay đổi nào gần đầu prefix đều invalidate mọi thứ phía sau.** Xem [per-turn-cost-math.md](./per-turn-cost-math.md) để biết điều đó thực sự tốn bao nhiêu.

---

## Fact 3 — Context to hơn không có nghĩa là focus tốt hơn

Self-attention — cơ chế model dùng để quyết định token nào quan trọng — gán cho mỗi token một trọng số, và **các trọng số đó phải cộng lại thành một tổng cố định**. Điều đó có nghĩa là mỗi token không liên quan thêm vào sẽ làm giảm trọng số dành cho những token liên quan. Nhiều context hơn làm signal mỏng đi, không sắc hơn.

Kết hợp với hiệu ứng "lost in the middle" đã được đo đạc thực nghiệm (thông tin bị vùi giữa prefix dài và suffix dài bị trọng số thấp hơn), một prompt dài gấp đôi có thể *kém hiệu quả hơn* một prompt ngắn bằng nửa, nếu phần nội dung thêm vào là noise. **Nhiều context hơn không phải là một bản nâng cấp miễn phí.**

Để có một giải thích kiểu analogy về cách self-attention thực sự hoạt động, vì sao 1M token là trần chứ không phải siêu năng lực, và nó có ý nghĩa gì cho việc viết prompt Claude Code, xem deep dive đi kèm: [self-attention.md](./self-attention.md).

---

## Điều này có ý nghĩa gì cho các primitive

Mọi primitive trong Claude Code là một lời giải cho cùng câu hỏi: *làm sao để feed model đúng cái nó cần, đúng lúc nó cần, mà không phải trả phí ở mỗi turn không dùng tới?*

- **`CLAUDE.md`, rule luôn-load, skill description:** trả phí *mỗi turn*. Làm phình chúng thì mọi response đều chậm hơn, đắt hơn, và kém focus hơn.
- **Skill body, rule path-scoped, memory topic file, subagent:** chỉ trả phí *khi liên quan*. Lối thoát cho nội dung nặng.
- **Hook:** trả phí *không gì* ở mức model. Enforcement sống hoàn toàn bên ngoài context window.

Một khi bạn thấm điều này, phần còn lại của README chính chỉ là áp dụng nguyên tắc vào các quyết định placement cụ thể.

---

## Related deep dives

- [`self-attention.md`](./self-attention.md) — cách attention thực sự hoạt động, vì sao window 1M-token không bằng 1M token hiệu dụng.
- [`per-turn-cost-math.md`](./per-turn-cost-math.md) — caching, invalidation, và một lần edit `CLAUDE.md` giữa session thực sự tốn bao nhiêu đô la trên 100 turn.
