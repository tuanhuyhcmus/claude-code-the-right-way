# Vì sao model to hơn không cứu được bạn — ba fact về LLM runtime

*Phụ lục cho [README.vi.md](../../README.vi.md). Câu trả lời cơ học cho câu hỏi "vì sao trần context là 1M chứ không phải 1T cho xong" — và vì sao một window to hơn cũng không thay đổi được hình dạng của vấn đề.*

Không cần hiểu sâu ML để đặt kiến thức đúng chỗ trong Claude Code. Bạn chỉ cần ba fact, ở dạng vừa đủ. Mọi quyết định placement trong README đều bắt nguồn từ ba fact này.

## Fact 1 — Context window có giới hạn, và được dùng chung

Opus 4.7 có giới hạn 1M token. Nghe khổng lồ (với người đã đọc [self-attention](./self-attention.vi.md) rồi thì thấy nó khổng lồ, chứ so 1M với dung lượng bộ nhớ, RAM... thì muỗi — nó mà đạt đến mức hàng TB như bộ nhớ thì giờ chúng ta đi làm nông dân hết rồi); thật ra không, vì budget được **dùng chung**, không dành riêng cho câu hỏi của bạn. Mọi thứ bên dưới tranh cùng 1M đó *ở mỗi turn*:

- System prompt của chính Claude Code
- Mọi tool schema — built-in, MCP, agent definition
- Mọi `CLAUDE.md`, mọi rule luôn-load, mọi skill description, `MEMORY.md`
- Toàn bộ hội thoại trước, mọi tool result, mọi file Claude đã đọc

Một lần đọc file source 2000 dòng có thể ăn 30–50k token. Mười lần đọc như vậy trong một session debug dài và bạn đã đốt 5% window trước khi làm việc thật. "1M token" là trần, không phải workspace — workspace dùng được co lại mỗi phút trong session.

## Fact 2 — Mỗi request gửi lại toàn bộ hội thoại

Window hữu hạn sẽ dễ chịu nếu model giữ được state giữa các turn. Nó không. Inference LLM là **stateless** — client gửi lại toàn bộ hội thoại trước, mọi `CLAUDE.md` đã load, mọi rule, mọi skill description, mọi tool definition, **mỗi turn**, từ turn 1.

Hệ quả thực tế: thêm 500 dòng vào `CLAUDE.md` không phải chi phí một lần. Nó là thuế mỗi turn, nhân với mọi turn của mọi session tương lai. Prompt caching làm nhẹ tác động (0.1× cho prefix khớp trong TTL 5 phút), nhưng caching chỉ giúp nếu bạn không invalidate prefix — và edit `CLAUDE.md` giữa session invalidate mọi thứ phía sau.

## Fact 3 — Context to hơn không có nghĩa là focus tốt hơn

Trả phí mỗi turn cũng sẽ dễ chịu nếu mỗi token bạn trả tiền đều làm việc đều nhau. Nó không. Self-attention — cơ chế model dùng để quyết định token nào quan trọng — gán cho mỗi token một trọng số, và **các trọng số phải cộng lại thành một tổng cố định**. Mỗi token không liên quan thêm vào làm giảm trọng số dành cho những token liên quan. Nhiều context hơn làm signal *mỏng đi*, không sắc hơn. Kết hợp với hiệu ứng "lost in the middle" đã được đo đạc thực nghiệm (thông tin bị vùi giữa prefix dài và suffix dài bị trọng số thấp hơn), một prompt dài gấp đôi có thể *kém hiệu quả hơn* một prompt ngắn bằng nửa, nếu phần thêm vào là noise.

> **Deep dive:** [`self-attention.vi.md`](./self-attention.vi.md) minh họa vì sao đây là vấn đề *cơ học*, không phải bug mà một window to hơn sẽ sửa. [`why-context-matters.vi.md`](./why-context-matters.vi.md) mở rộng cả ba fact với ví dụ cụ thể về cái giá mỗi fact bắt bạn trả trong thực tế.

## Quỹ đạo có đáy

Ba fact này không được fix bởi một window to hơn. Chúng *cộng dồn* với nó. Model sẽ tiếp tục thông minh lên; context window tiếp tục to ra — 8k → 200k → 1M → cái tiếp theo. Các thắng lợi kỹ thuật như prompt caching, path-scoped rule, compaction, subagent isolation đều đẩy trần lên. Nhưng chúng không đổi *hình dạng* của trần: **vấn đề scale theo tham vọng của bạn, không theo model.** Ngày window 10M-token đến, bạn sẽ muốn nhét 20M token context project vào đó.

## Reasoning không phải là durable knowledge

Thêm một fact nữa, lần này là quan sát thực tế. Tôi từng apply một config Terraform và fail. Không cần hand-holding, Claude đoán URL có thể sai, đi tìm source của provider, không tìm được local (provider được đặt tên trái quy ước), web-search tên gần nhất, tìm thấy GitHub repo, đọc source, và phát hiện một dấu `/` thừa không nên có ở cuối. Mind-blowing. Và tất nhiên sau đó tôi không dám đóng session — chắc chắn cái expertise đó đang sống *đâu đó*. Đúng là sau N turn làm việc không liên quan, một vấn đề tương tự xuất hiện và Claude phải suy luận lại một nửa từ đầu.

Một model đủ thông minh để *suy ra* câu trả lời đúng một lần vẫn sẽ suy ra lại, *không hoàn hảo*, mỗi khi context của nó drift hay reset. Trong session dài, attention rot dần dần chôn vùi câu trả lời. Trong session mới, nó chưa bao giờ có ở đó.

> **Deep dive:** [`delegation-ceiling.vi.md`](./delegation-ceiling.vi.md) cho câu chuyện đầy đủ và vì sao chỉ model tốt hơn không thu hẹp được khoảng cách.
