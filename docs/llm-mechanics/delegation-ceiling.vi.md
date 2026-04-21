# The delegation ceiling

*[English](./delegation-ceiling.md) · **Tiếng Việt***

*Bài đọc bổ sung tùy chọn cho README chính. Delegation "ma thuật" thực sự trông thế nào khi nó hoạt động — và vì sao vấn đề nền bên dưới không biến mất dù model, context window, và infrastructure có tiếp tục cải thiện đến đâu.*

Phần mở đầu của README chính mặc định một vài khẳng định: rằng lời khuyên "delegate, don't dictate" thực sự hoạt động ở cấp một prompt duy nhất, rằng quỹ đạo của model là có thật, và rằng dù vậy, vẫn có một cái sàn mà quỹ đạo đó không vượt qua được. Người đọc đã tin rồi có thể bỏ qua file này. Người đọc muốn một câu chuyện cụ thể và một lần kiểm chứng sâu hơn với thực tế thì đọc tiếp.

---

## Smart reasoning không thay thế được durable knowledge

Khi loop siết lại, kết quả có thể thực sự ma thuật. Một ví dụ thật:

Tôi từng apply một config Terraform để tạo một server. Nó fail. Không cần hand-holding gì hết, Claude:

1. Suy ra URL có thể là vấn đề,
2. Đi đọc source của provider để verify — nhưng provider được đặt tên trái quy ước trong file `.tf` của tôi, nên Claude không tìm được local,
3. Web-search tên khớp gần nhất,
4. Tìm ra đúng GitHub repo, đọc source,
5. Phát hiện URL trong config của tôi có một dấu `/` thừa không nên có. Fix luôn.

Thực sự kinh khủng. Và tất nhiên sau đó tôi không dám đóng session — chắc chắn cái expertise đó đang sống *đâu đó* trong đây. Đúng như dự đoán, N turn làm việc không liên quan sau đó, một vấn đề tương tự xuất hiện và Claude phải suy luận lại một nửa từ đầu.

**Bài học:** reasoning không giống với durable knowledge. Một model đủ thông minh để *suy ra* câu trả lời đúng một lần vẫn sẽ suy ra lại, không hoàn hảo, mỗi khi context của nó drift hay reset. Trong một session dài, attention rot dần dần chôn vùi câu trả lời. Trong một session mới, nó chưa bao giờ có ở đó. Bạn cần một cách để khiến kết luận *dính lại.*

---

## Quỹ đạo thực sự có giúp

Model ngày càng thông minh hơn. Context window ngày càng lớn ra — 8k → 200k → 1M → bất kể cái tiếp theo là gì. Infrastructure xung quanh model cũng ngày càng tốt hơn — cache thông minh hơn, nén hội thoại cũ tốt hơn, cách cách ly công việc nặng gọn gàng hơn. Có lý khi giả định trải nghiệm dễ thở hơn một chút mỗi năm, và rất nhiều cạnh gai mà người ta đang đụng hôm nay sẽ mềm đi theo thời gian.

---

## Nhưng quỹ đạo có một cái sàn

Chừng nào hệ thống nền bên dưới vẫn là một **LLM được driven bởi self-attention**, ba điều sau vẫn đúng bất kể window to đến đâu:

1. Window vẫn hữu hạn.
2. Mỗi token vẫn cạnh tranh cho một attention budget cố định (nên càng nhiều context thì signal trên mỗi token càng mỏng — hiệu ứng "lost-in-the-middle" không biến mất ở 10M token cũng như nó đã không biến mất ở 8k).
3. Inference vẫn stateless — toàn bộ hội thoại re-send ở mỗi turn.

Các cải tiến kỹ thuật đẩy trần lên; chúng không đổi hình dạng của trần. Compaction mua cho bạn một session hiệu dụng lớn hơn, nhưng nó mua bằng cách *vứt bỏ nội dung*, không phải nhớ được nhiều hơn. Prompt caching làm re-send rẻ, nhưng re-send vẫn phải vừa window. **Vấn đề scale theo tham vọng của bạn, không theo model** — ngày một window 10M-token đến, bạn sẽ thấy mình muốn nhét 20M token context project vào đó.

---

## Related deep dives

- [`why-context-matters.md`](./why-context-matters.md) — ba fact về LLM runtime chi tiết hơn, với ví dụ về cái giá mỗi fact bắt bạn trả trong thực tế.
- [`self-attention.md`](./self-attention.md) — vì sao hiệu ứng "lost-in-the-middle" là cơ học, không phải bug mà một window to hơn sẽ sửa.
- [`per-turn-cost-math.md`](./per-turn-cost-math.md) — caching, compaction, và một lần edit `CLAUDE.md` giữa session thực sự tốn bao nhiêu.
