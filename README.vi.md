# Claude Code, đúng cách

*[English](./README.md) · **Tiếng Việt***

Một hướng dẫn có định kiến rõ ràng về việc **tổ chức kiến thức** trong một dự án Claude Code — để bạn thoát khỏi việc cộng tác với Claude theo cảm tính và bắt đầu định hình context một cách có chủ đích.

Mục tiêu không phải là biến Claude Code thành một cỗ máy sinh code hào nhoáng — kiểu template engine, hay autocomplete thêm mấy bước thừa — nơi mọi câu trả lời đều được định sẵn bởi scaffolding của bạn và model không còn chỗ để suy nghĩ. Mục tiêu ngược lại: **cho Claude đủ structure bền vững để những phần thông minh, khám phá của model thực sự rơi vào một chỗ có thể dự đoán được**, thay vì phải tự suy ra lại những fact cơ bản về codebase của bạn hết session này đến session khác. Một skill, một rule, một hook không phải thứ thay thế cho khả năng lập luận của model — nó là cái giàn đỡ xung quanh. Làm đúng, mọi primitive trong hướng dẫn này sẽ làm model *hữu ích hơn*, không kém đi: đáng tin ở những chỗ cần đáng tin, tự do suy nghĩ ở những chỗ không nên bị gò.

> Đúc kết từ trải nghiệm hằng ngày trên một Go monorepo cỡ vừa. YMMV — coi đây là điểm khởi đầu, đừng coi là giáo điều.

Trước khi bàn *cách* tổ chức, nên thành thật một chút về chuyện gì sẽ hỏng nếu không tổ chức — vì đó là tình trạng của đa số session Claude Code hiện nay.

---

## 1. "Delegate, don't dictate" âm thầm đổ vỡ trong session dài

Cách đa số người dùng Claude Code hiện nay là **giao phó ý định (delegating intention)** — và chính docs của Anthropic tích cực khuyến khích điều này. Lời khuyên của họ rất rõ: *["Delegate, don't dictate. Think of delegating to a capable colleague. Give context and direction, then trust Claude to figure out the details."](https://code.claude.com/docs/en/how-claude-code-works#delegate-dont-dictate)* Lời khuyên đó đúng ở đúng mức nó được viết ra cho: **một prompt duy nhất đủ sức tự mô tả trọn vẹn vấn đề** — đủ context, đủ direction, đủ signal để Claude có cái nó cần mà không phải được chỉ đọc file nào hay chạy command nào. Khi prompt đạt ngưỡng đó, lời khuyên chính xác không thể cãi: bạn ngừng quản lý vi mô, model chọn đường, loop siết lại.

Repo này không phải để phản biện lời khuyên đó. Nó bàn về *những gì lời khuyên kia không nói tới*: điều gì xảy ra khi **mọi session trong nhiều tháng liền** đều chạy kiểu này — khi delegation không còn là lối viết prompt mà là cả workflow, không có gì ở giữa hai cực *"tôi gõ vài chữ mơ hồ"* và *"Claude tự tìm ra"*.

Giờ để ý xem lời khuyên đó *ngầm giả định* điều gì. *"Give context and direction"* là một **lời hứa có trạng thái (stateful promise)** — context và direction bạn đưa ở turn 1 phải còn hiệu lực khi Claude ra tay ở turn 47. Lời hứa đó không sống sót qua session dài, vì ngay khi cuộc trò chuyện to đến mức căng context window, Claude Code không từ chối tiếp tục — nó kích hoạt **compaction**, và âm thầm viết lại những turn cũ thành một bản tóm tắt để session còn chạy được.

Nói nhanh về `/compact`, vì mọi thứ bên dưới đều dựa vào nó. Khi một session Claude Code tiến sát trần context (1M token với Opus 4.7), client sẽ chạy **compaction**: các turn cũ — prompt cũ của bạn, mọi tool result, mọi file đã đọc, mọi reply Claude viết — bị **model tự tóm tắt thành một khối có cấu trúc ngắn**. Nôm na, vài trăm nghìn token lịch sử nguyên văn sụp xuống còn vài chục nghìn token tóm tắt (khoảng ~800k → ~100–200k tùy nội dung). Phần nguyên văn biến mất; chỉ bản tóm tắt sống tiếp sang turn sau. Đây là thứ giữ cho session dài không chết, và cũng là nơi phần lớn context bạn cẩn thận đưa vào *chết âm thầm*.

Sau một lần `/compact`, direction ban đầu của bạn không còn là instruction nguyên văn nữa — nó là một vài câu nằm trong bản tóm tắt do model tự viết. Sau ba chục lần `/compact` trong một tuần, nó là một đống tóm-tắt-của-tóm-tắt có thể không còn khớp với nhau. Bạn không thể đòi Claude, sau lần compaction thứ 30, nhớ lại toàn bộ ý đồ bạn đặt ra trước lần đầu tiên. Docs im lặng về chuyện này.

Lối thoát lộ liễu là đóng session rồi mở session mới. Cách đó không sửa được vấn đề — nó chỉ *đổi hình dạng của việc quên*. `/compact` để lại cho bạn một bóng ma đã bị diễn giải lại của context ban đầu; session mới để lại cho bạn *không gì cả*. Cả hai đẩy Claude về cùng một chỗ: *nó không còn biết hôm qua nó đã biết gì.* Không cái nào là workflow đáng tin.

Câu hỏi thật tình phải đặt ra là: *bạn có thực sự biết Claude đang chú ý vào cái gì ngay lúc này không?* Nếu thành thật, câu trả lời thường là "theo cảm giác". Và cảm giác mơ hồ đó biểu hiện qua vài triệu chứng rất cụ thể mà bất kỳ ai dùng Claude Code đã lâu đều nhận ra:

- **Session anxiety — nỗi lo session.** Bạn làm với Claude được 90 phút. Mọi thứ đang trôi. Bạn không dám đóng session. Không phải vì biết Claude sẽ quên cái gì cụ thể — mà vì *bạn không biết nó đang nhớ cái gì, điểm.* Với đầu óc kỹ sư, đó là loại dependency tệ nhất: **không biết cái mình không biết**.
- **Nhồi nhét (cramming).** Claude vừa fix một edge case. Bạn lập tức cố fix hết mọi edge case tương tự trong cùng session — hoặc hí húi viết docs, thầm cầu nguyện *"lần sau Claude sẽ đủ khôn để nhớ"*. Bạn làm vậy không vì đó là workflow đúng. Bạn làm vì không tin session mới sẽ rơi vào đúng trạng thái attention này.
- **Mê tín workflow.** Claude vừa chạy xong setup integration-test 5 bước: *spin up env → khởi động service phụ thuộc → override local config → chạy test case → in báo cáo dễ đọc*. Mọi thứ khớp. Bạn ghi nhận, với chút khó chịu, rằng bạn rất không muốn làm lại từ đầu — dù về mặt logic, toàn bộ state đó nằm trong file và shell command, không phải trong đầu Claude.
- **Nghĩa địa `CLAUDE.md`.** Sau mỗi lần chỉnh đau đớn, bạn nhét thêm một bullet vào `CLAUDE.md`. Sáu tháng sau, file dài 400 dòng và bạn không còn chắc bullet nào đang thực sự kích hoạt, bullet nào mâu thuẫn với bullet nào, hay bullet nào đã hết đúng từ hai sprint trước.

Cả bốn triệu chứng là cùng một lời than trong bốn hình dạng khác nhau: **bạn không biết Claude đang biết gì, ngoài cảm giác.** Tin tốt là cảm giác đó giải quyết được — mọi thứ Claude đang mang theo đều là file trên đĩa, và hai câu lệnh sẽ chỉ chúng cho bạn.

---

## 2. Nhìn xem Claude đang thực sự mang theo cái gì

Bắt đầu ở đây. Mở một session Claude Code và gõ:

```text
/memory
```

Bạn sẽ thấy mọi `CLAUDE.md`, mọi rule đã load, và mọi auto-memory file đang nằm trong context của session này. Với đa số người dùng Claude Code được vài tháng, lần `/memory` đầu tiên là một cú sốc nhẹ. Có auto-memory được viết trong lúc bực bội cách đây nhiều tháng. Có những fact về project đã hết đúng từ hai sprint trước. Có rule mà Claude vẫn ngoan ngoãn tuân theo nhưng không ai trong team nhớ đã viết ra.

Còn một ghi chép thứ hai cũng đáng xem: **transcript hội thoại** của bạn. Mọi trao đổi bạn từng có với Claude đều nằm trên đĩa — một kho lưu trữ đọc được về cách hai bên cộng tác. Bạn có thể tự đọc, hoặc thú vị hơn, đưa cho một LLM khác đọc rồi hỏi xem prompt của bạn hay mơ hồ chỗ nào, hay phải giải thích lại cái gì, hay Claude cứ sai lại sai cùng một kiểu. Transcript là thứ gần nhất với một *nhật ký luyện tập* khi làm việc với LLM.

Không phải dòng nào trong `/memory` cũng cùng chi phí, và không phải dòng nào cũng sống sót qua cùng loại sự kiện. Các mục có lifecycle khác nhau rõ rệt: cái gì prompt cache lưu, cái gì `/compact` giữ lại hay bỏ đi, cái gì reload giữa session tùy theo file Claude đọc. Có hai hệ quả không hiển nhiên đáng nhớ trước khi đi tiếp:

- **Skill description là khối luôn-được-load duy nhất bị `/compact` âm thầm bỏ.** `/command` vẫn chạy sau compaction (harness load skill body trực tiếp từ đĩa), nhưng auto-trigger *implicit* của Claude không còn thấy bất cứ skill nào chưa từng được invoke trong session này — lý do vì sao các framework auto-triggering "magic" rõ ràng đần đi sau session dài.
- **Path-scoped rule và nested `CLAUDE.md` là những mục dễ gãy nhất trong `/memory`.** Chúng nằm trong message history, nên compaction xóa sạch, và chỉ quay lại khi Claude tình cờ đọc lại một file khớp path.

Bảng lifecycle đầy đủ — mục nào sống qua `/compact`, mục nào được prefix-cached, mục nào reload theo yêu cầu — xem [`docs/llm-mechanics/memory-entry-lifecycle.md`](./docs/llm-mechanics/memory-entry-lifecycle.md).

### Cái nhìn budget: `/context`

`/memory` cho biết *file nào* đã load. Nó không cho biết *tốn bao nhiêu*. Cho việc đó, có câu lệnh thứ hai, được cho là hữu ích hơn:

```text
/context
```

Nó đưa ra phân nhóm chi tiết token budget của bạn đã đi đâu. Ví dụ thật (số từ một session thật, biên tập gọn):

```text
Context Usage — 200k / 1M tokens (20%)

  System prompt:      9.0k   ( 0.9%)
  System tools:      13.8k   ( 1.4%)
  Custom agents:      0.4k   ( 0.0%)
  Memory files:       4.8k   ( 0.5%)
  Skills:             2.0k   ( 0.2%)
  Messages:         172.5k   (17.2%)
  Autocompact buf.:  33.0k   ( 3.3%)
  Free space:       764.6k   (76.5%)
```

Dưới bản tóm tắt, `/context` drill xuống từng category và liệt kê **từng file, từng tool** đóng góp vào budget — không chỉ tổng theo nhóm. Cái drill xuống từng item đó là câu trả lời cho câu hỏi "context của tôi thực sự đi đâu?" mà riêng `/memory` không trả lời được:

- Một `CLAUDE.md` phì đại xuất hiện thành một dòng *béo* dưới *Memory files*. (Trong session ở trên, một `.claude/rules/integration-test-common.md` 3.2k-token chiếm hai phần ba toàn bộ memory budget.)
- Mỗi MCP server bạn bật sẽ đẩy *System tools* lên — và đẩy ở *mỗi turn, mãi mãi*, dù bạn có dùng tool đó hay không.
- Hội thoại dài thổi phồng *Messages* — đây là chi phí tích lũy của tool result, file đọc vào, và turn trước. Dòng này luôn lớn nhanh nhất.
- *Autocompact buffer* là phần đệm Claude Code dành sẵn để `/compact` có chỗ chạy. Khi *Free space* tiến gần buffer đó, auto-compaction sẽ tự động kích hoạt, dù bạn có muốn hay không.

Một câu chuyện cảnh báo về việc *Memory files* có thể tệ đến đâu. Một người bạn từng chạy `/init` khi đang đứng trong một directory chứa **100 project**. Claude ngoan ngoãn scan hết từng project và tạo ra một `CLAUDE.md` quái vật cỡ **~1 triệu token** — nghĩa là mọi project trong folder đó bị flatten thành context-priors. Từ đó trở đi, mọi session mới mở ra đều đã đầy ~95% window 1M-token *trước khi anh ta gõ phím*. Response chậm, đắt, và hơi loạn, và gần trọn một ngày anh ta không biết tại sao. Đó là chuyện xảy ra khi `CLAUDE.md` rơi vào sai cấp directory: nó không còn là file config nữa mà thành thuế định kỳ mỗi turn, lớn hơn cả cuộc hội thoại thật. Luôn scaffold `/init` ở *project root*, đừng bao giờ ở folder-chứa-nhiều-project.

**Insight có thể hành động:** nếu *Messages* chiếm ưu thế, một lần `/compact` hoặc session mới sẽ giải quyết. Nếu *System tools*, *Memory files*, hoặc *Skills* to không tương xứng, bạn có **vấn đề structural** — đổi session không cứu được — bạn phải cắt bớt MCP server, xóa rule lỗi thời, hoặc đánh dấu skill là `disable-model-invocation: true`.

#### Mấy con số token này có thật không? Kiểm chéo ba hướng

Hoài nghi hợp lý: liệu `/context` có đang hiển thị số "minh họa" không phản ánh cái thực sự gửi đến Anthropic? Ba nguồn độc lập nói không — mọi token ở đó là token thật trong request thật:

1. **Docs Anthropic** nói thẳng: *["Skill descriptions are loaded into context so Claude knows what's available."](https://code.claude.com/docs/en/skills)* Tương tự cho CLAUDE.md, rule, và auto-memory.
2. **Session log chứa nguyên văn.** Session JSONL tại `~/.claude/projects/<project>/<session-id>.jsonl` chứa một `skill_listing` attachment ở turn đầu, với **toàn bộ text** của mọi skill description nối lại. Chia độ dài ký tự cho ~4 thì khớp với con số "Skills" mà `/context` báo. Các attachment tương tự (`nested_memory`, `deferred_tools_delta`) ghi lại nội dung của mọi rule, CLAUDE.md, và MCP tool list đã load.
3. **Accounting API khớp.** Giá trị `cache_creation_input_tokens` của assistant message đầu tiên bằng tổng của toàn bộ category startup trong `/context` cộng một phần framing overhead nhỏ. Nếu một category là giả, con số này sẽ thấp hơn đúng bằng kích thước category đó.

**Vì sao quan trọng (và nó tốn bạn bao nhiêu):**

- Mỗi skill, mỗi rule, mỗi MCP tool đã load đều tốn token **ở mỗi turn**, không chỉ khi được invoke. Cache hit làm turn lặp rẻ (0.1× giá base input), nhưng bất kỳ cache miss nào — idle quá 5 phút TTL mặc định, edit `CLAUDE.md`, thêm skill mới — đều viết lại toàn bộ prefix với giá 1.25× base.
- Một framework kiểu *"50 skill trong một repo"* × ~100 token mỗi description = **~5k token mỗi turn, mãi mãi**, dù bạn có dùng skill đó hay không. Đây không phải chi phí cài đặt một lần; đây là thuế định kỳ trên mọi session.
- `disable-model-invocation: true` *thực sự* đưa skill ra khỏi khoản thuế đó. Description của nó không bao giờ vào startup prompt, nên tốn 0 token ở những turn bạn không invoke bằng `/name`.

> **Deep dive:** [`docs/llm-mechanics/per-turn-cost-math.md`](./docs/llm-mechanics/per-turn-cost-math.md) — bảng giá đầy đủ. Ví dụ thực tế cho biết một lần edit `CLAUDE.md`, một lần pha cà phê 10 phút, hay một MCP server phì đại thực sự tốn bao nhiêu cho 100 turn, với số Opus 4.7. Có phân tích break-even cho tier cache 1 giờ.

Mối quan hệ trung thực giữa hai câu lệnh:

- **`/context`** là khung chẩn đoán toàn diện — mọi category nội dung hiện đang (hoặc sẽ) vào context, có chi phí token, drill theo item. Tất cả những gì `/memory` hiện là tập con của `/context`, *cộng* token, *cộng* skill, agent, MCP tool, và phần accounting Messages / Free-space / Autocompact-buffer.
- **`/memory`** là editor cho memory file. Nó liệt kê cùng bộ `CLAUDE.md`, rule, auto-memory, nhưng nhiệm vụ của nó là cho bạn *sửa* — chọn một file thì mở trong editor, hoặc bật/tắt auto-memory. Để chẩn đoán, `/context` mạnh hơn hẳn; `/memory` thắng khi bạn muốn sửa ngay một file vừa phát hiện.

Nếu chỉ học một, học `/context`. Nó là view thực tế nhất để hiểu tại sao một session dài chậm, phân tâm, hay đắt — và là chỗ đầu tiên nên nhìn trước khi đổ lỗi cho model. Chạy nó mỗi 20–30 turn trong session dài; nhìn Messages tăng trong khi Memory / Skills / Tools đứng yên sẽ dạy bạn, bằng trực giác, phần nào trong budget nằm trong kiểm soát của bạn và phần nào không.

`/memory` và `/context` cho biết *cái gì* Claude đang mang, và *tốn gì*. Chúng không thể cho biết *tại sao* phải cắt giảm, hay tại sao context window to hơn không phải là giải pháp thay thế. Cả hai câu hỏi đó có chung một câu trả lời — ba fact về cách LLM thực sự chạy tại inference time.

---

## 3. Vì sao model to hơn không cứu được bạn — ba fact về LLM runtime

Không cần hiểu sâu ML để đặt kiến thức đúng chỗ trong Claude Code. Bạn chỉ cần ba fact, ở dạng vừa đủ. Mọi quyết định placement ở phần còn lại của hướng dẫn này chảy ra từ chúng.

### Fact 1 — Context window có giới hạn, và được dùng chung

Opus 4.7 trần 1M token. Nghe khổng lồ. Thật ra không, vì budget được **dùng chung**, không dành riêng cho câu hỏi của bạn. Mọi thứ bên dưới tranh cùng 1M đó *ở mỗi turn*:

- System prompt của chính Claude Code
- Mọi tool schema — built-in, MCP, agent definition
- Mọi `CLAUDE.md`, mọi rule luôn-load, mọi skill description, `MEMORY.md`
- Toàn bộ hội thoại trước, mọi tool result, mọi file Claude đã đọc

Một lần `Read` file source 2000 dòng có thể ăn 30–50k token. Mười lần đọc như vậy trong một session debug dài là bạn đã đốt 5% window trước khi làm việc thật. "1M token" là trần, không phải workspace — workspace dùng được co lại mỗi phút trong session.

### Fact 2 — Mỗi request gửi lại toàn bộ hội thoại

Window hữu hạn sẽ dễ chịu nếu model giữ được state giữa các turn. Nó không. Inference LLM là **stateless** — client gửi lại toàn bộ hội thoại trước, mọi `CLAUDE.md` đã load, mọi rule, mọi skill description, mọi tool definition, **mỗi turn**, từ turn 1.

Hệ quả thực tế: thêm 500 dòng instruction vào `CLAUDE.md` không phải chi phí một lần. Nó là thuế mỗi turn, nhân với mọi turn tương lai của mọi session tương lai. Prompt cache giảm bớt (0.1× cho prefix khớp trong TTL 5 phút), nhưng cache chỉ giúp nếu bạn không invalidate prefix — và edit `CLAUDE.md` giữa session invalidate mọi thứ phía sau.

### Fact 3 — Context to hơn không có nghĩa là focus tốt hơn

Trả phí mỗi turn cũng sẽ dễ chịu nếu mỗi token bạn trả làm việc đều nhau. Nó không. Self-attention — cơ chế model dùng để quyết định token nào quan trọng — gán cho mỗi token một trọng số, và **các trọng số phải cộng lại thành một tổng cố định**. Mỗi token không liên quan thêm vào làm giảm trọng số dành cho những token liên quan. Nhiều context hơn làm signal *mỏng đi*, không sắc hơn. Kết hợp với hiệu ứng "lost in the middle" đã được đo đạc (thông tin bị vùi giữa prefix dài và suffix dài bị trọng số thấp hơn), một prompt dài gấp đôi có thể *kém hiệu quả hơn* một prompt ngắn bằng nửa, nếu phần thêm vào là noise.

> **Deep dive:** [`docs/llm-mechanics/self-attention.md`](./docs/llm-mechanics/self-attention.md) minh họa vì sao đây là vấn đề *cơ học*, không phải bug mà một window to hơn sẽ sửa. [`docs/llm-mechanics/why-context-matters.md`](./docs/llm-mechanics/why-context-matters.md) mở rộng cả ba fact với ví dụ cụ thể về giá của mỗi fact.

### Quỹ đạo có đáy

Ba fact này không được fix bởi một window to hơn. Chúng *cộng dồn* với nó. Model sẽ tiếp tục thông minh lên; context window tiếp tục to ra — 8k → 200k → 1M → cái tiếp theo. Các thắng lợi kỹ thuật như prompt caching, path-scoped rule, compaction, subagent isolation đều đẩy trần lên. Nhưng chúng không đổi *hình dạng* của trần: **vấn đề scale theo tham vọng của bạn, không theo model.** Ngày window 10M-token đến, bạn sẽ muốn nhét 20M token context project vào đó.

### Reasoning không phải là durable knowledge

Thêm một fact nữa, lần này là quan sát thực tế. Tôi từng apply một config Terraform và fail. Không cần hướng dẫn, Claude đoán URL có thể sai, đi tìm source provider, không tìm được local (provider được đặt tên trái quy ước trong file `.tf`), web-search tên gần nhất, tìm thấy GitHub repo, đọc source, và phát hiện URL có một dấu `/` thừa không nên có. Fix luôn. Kinh khủng thật. Và tất nhiên sau đó tôi không dám đóng session — chắc chắn cái expertise đó đang sống *đâu đó* trong đây. Đúng là sau N turn làm việc khác, một vấn đề tương tự xuất hiện và Claude phải suy luận lại một nửa từ đầu.

Một model đủ thông minh để *suy ra* câu trả lời đúng một lần vẫn sẽ suy ra lại, *không hoàn hảo*, mỗi khi context của nó drift hay reset. Trong session dài, attention rot dần dần chôn vùi câu trả lời. Trong session mới, nó chưa bao giờ có ở đó.

> **Deep dive:** [`docs/llm-mechanics/delegation-ceiling.md`](./docs/llm-mechanics/delegation-ceiling.md) cho câu chuyện đầy đủ và vì sao model tốt hơn không thu hẹp được khoảng cách.

Gộp lại: window hữu hạn, mỗi token thêm vào làm loãng mọi token khác, và những kết luận mà session thông minh *suy ra* bị suy ra lại — không hoàn hảo — bởi session tiếp theo phải bắt đầu từ đầu. Miếng ghép còn thiếu là một chỗ để làm cho những kết luận đó **dính lại** — một chỗ `/compact` không đụng được và session mới không thể không thấy. Đó chính xác là mục đích các primitive của Claude Code.

---

## 4. Các primitive — chỗ bền vững nằm ngoài `/compact`

Sáu primitive, mỗi cái trả lời một biến thể của cùng một câu hỏi: *đặt cái này vào đâu để nó sống qua `/compact`, và chỉ vào lại context ở đúng những turn cần?*

- **`CLAUDE.md` và rule** đóng gói kiến thức project bền vững và đưa vào context ở mỗi turn liên quan. Body là instruction; Claude đọc thụ động.
- **Skill và subagent** đóng gói kiến thức *thủ tục* — *"khi thấy X, làm Y₁ → Y₂ → Y₃"* — và offer cho model kèm metadata (`description`, `when_to_use`, `paths`) về lúc nên với tới. Body không vào context cho đến khi được invoke. Subagent còn isolate công việc của mình trong một context window mới, nên cuộc hội thoại chính không bị ô nhiễm.
- **Hook** thực thi số ít invariant mà không thể tin model tự giữ. Chúng chạy hoàn toàn bên ngoài model, nên miễn nhiễm với attention rot, context drift, hay một prompt khéo léo thuyết phục Claude đi ngược.
- **Memory file** lưu lại fact user-local xuyên session — những thứ về bạn hay project thay đổi theo thời gian nhưng không nên gõ lại mỗi session.

<details>
<summary>Mở ra nếu cần nhắc lại các tên này.</summary>

**`CLAUDE.md`** — một file markdown ở root repo (hoặc root subdirectory) mang **project context bền vững** Claude nên biết ở đầu mỗi session: build command, quy ước, ghi chú kiến trúc, quy tắc đặt tên, workflow thường dùng, những thứ bạn ngán phải giải thích lại. Claude đọc mỗi turn, nên đây cũng là chỗ đặt behavioral guideline ("ưu tiên edit thay vì tạo file mới", "đừng thêm comment trừ khi được yêu cầu"). Có thể scaffold file ban đầu bằng `/init` trong project. Một `CLAUDE.md` lỗi thời, mâu thuẫn, hay dài 400 dòng là nguồn gốc số một của lời than "Claude Code cứ làm sai". [Docs chính thức](https://docs.claude.com/en/docs/claude-code/memory).

**Rule** (`.claude/rules/*.md`) — file markdown dưới `.claude/rules/` mà Claude Code tự discover đệ quy và load với priority ngang `.claude/CLAUDE.md`. Dùng để tách invariant project ("MUST / MUST NOT") ra khỏi một `CLAUDE.md` phì đại. YAML `paths:` frontmatter scope rule vào pattern file cụ thể để nó chỉ load khi Claude đụng file khớp. Rule mức user ở `~/.claude/rules/`. [Docs chính thức](https://code.claude.com/docs/en/memory#organize-rules-with-claude/rules/).

**Skill** (`.claude/skills/<name>/SKILL.md`) — workflow được đặt tên và trigger-based mà Claude có thể invoke. Mỗi skill có description nói Claude khi nào dùng, cùng các bước procedural bên trong. Lý tưởng cho "khi user nói X, làm Y₁ → Y₂ → Y₃". [Docs chính thức](https://docs.claude.com/en/docs/claude-code/skills).

**Hook** (`.claude/hooks/*.sh` + `settings.json`) — shell script mà harness Claude Code chạy tự động ở các sự kiện lifecycle (pre-tool, post-tool, user-prompt-submit, v.v.). Exit code khác 0 có thể chặn action. Đây là layer *thực thi bằng máy* — thứ duy nhất chặn Claude vi phạm rule *kể cả khi* rule đã nằm trong context. [Docs chính thức](https://docs.claude.com/en/docs/claude-code/hooks).

**Memory** (`~/.claude/projects/<project>/memory/*.md`) — local theo user, lưu xuyên session. Claude tự viết vào đây để nhớ fact về bạn, project, và preference. *Không* dành cho những thứ code hay `git log` trả lời được. [Docs chính thức](https://docs.claude.com/en/docs/claude-code/memory).

**Subagent** (invoke qua tool `Agent`) — một Claude instance mới với context window riêng, spawn ra để xử lý một task có phạm vi. Parent chỉ thấy summary cuối cùng. Dùng để cách ly việc đọc nặng (hàng nghìn dòng) hoặc công việc song song độc lập. [Docs chính thức](https://docs.claude.com/en/docs/claude-code/sub-agents).

</details>

### Tổng quan

| Primitive | Bản chất | Thời hạn | Phạm vi | Do ai thực thi |
|---|---|---|---|---|
| `CLAUDE.md` | Behavioral guideline (văn xuôi) | Committed | Repo / dir | Claude đọc mỗi turn |
| `.claude/rules/*.md` | Invariant khai báo (MUST / MUST NOT) | Committed | Repo | Claude + hook |
| `.claude/skills/*/SKILL.md` | Procedural workflow (how-to) | Committed | Repo / user | Claude invoke khi trigger |
| `.claude/hooks/*.sh` | Thực thi bằng máy | Committed | Repo | Harness (pre/post tool) |
| `memory/*.md` | State tạm của user / project | User-local | Xuyên session | Claude tự load |
| Subagent | Cách ly context | Theo lần invoke | Task-scoped | Tool `Agent` |

Mỗi primitive trả lời một câu hỏi khác nhau. Nhầm lẫn giữa chúng là sai lầm phổ biến nhất.

Đặt tên sáu primitive là phần dễ. Phần khó là biết *cái nào* khớp với một mẩu kiến thức cụ thể — và nhận ra rằng chọn sai không phải vấn đề phong cách, mà là một khoản thuế định kỳ trên mọi turn tương lai của mọi session tương lai. Phần còn lại của hướng dẫn là mental model cho việc đặt đâu.

---

## 5. Placement — ba trục, hai kiểu trigger, một decision tree

Các quyết định placement trở nên dễ hơn nhiều khi bạn thấm việc các primitive này nằm trên **ba trục khác nhau**, không phải một.

### Ba trục chi phí

**Trục 1 — Thuế context mỗi turn.** Những cái này vào context Claude ở đầu mỗi cuộc trò chuyện, dù bạn có dùng hay không:

- `CLAUDE.md` — toàn bộ nội dung
- `.claude/rules/*.md` không có `paths:` frontmatter — toàn bộ nội dung
- **Description** của skill (YAML frontmatter) — chỉ metadata, nhưng load cho *mọi* skill để Claude biết khi nào trigger
- **Description** của subagent — tương tự
- `MEMORY.md` — 200 dòng / 25KB đầu

Ngân sách tính sao cho hợp. Một repo 50 skill × 20 dòng description = ~1000 dòng thuế trả trước, trước khi bạn gõ một chữ.

**Trục 2 — Bổ sung prompt theo yêu cầu.** Những cái này chỉ vào context khi Claude quyết định chúng liên quan:

- **Body** của skill (mọi thứ trong `SKILL.md` sau frontmatter) — load khi Claude invoke
- **Execution context** của subagent — isolate trong window mới; parent chỉ thấy summary cuối
- Rule có path-scope (`paths: src/api/**`) — load khi Claude đọc file khớp
- Memory topic file (mọi file trong `memory/` trừ `MEMORY.md`) — load theo yêu cầu
- `CLAUDE.md` nested trong subdirectory — load khi Claude đụng file trong đó

Đây là chỗ của nội dung nặng, đặc thù task. Miễn khỏi budget "mỗi turn".

**Trục 3 — Hoàn toàn bên ngoài model.** Hook là loại khác hẳn. Chúng chạy trong harness Claude Code (client Node.js), không trong model:

- Không tốn context token
- Deterministic: exit code của shell chặn / cho qua; model không override được
- Không thể "quên" giữa hội thoại
- Bị giới hạn ở những gì một shell command thực sự có thể verify

Hook là layer duy nhất biến "MUST NOT" từ hy vọng thành thực thi. Mọi primitive khác đều phụ thuộc việc Claude đọc, hiểu, và chọn tuân theo.

### Ranh giới tin cậy trong skill và subagent

Hai trong ba trục sống *bên trong* skill và subagent, vì các primitive đó có một tính chất tách đôi mà các primitive khác không có: nửa metadata sống ở Trục 1, nửa body sống ở Trục 2.

|                 | Khi nào Claude thấy         | Để làm gì                         |
| --------------- | --------------------------- | --------------------------------- |
| **Description** | Mỗi turn (Trục 1)           | Routing: *"có nên invoke không?"* |
| **Body**        | Chỉ sau khi invoke (Trục 2) | Execution: *"giờ làm gì?"*        |

Claude route chỉ dựa trên description. Body không có chỗ để "phản đối" cho đến khi quyết định invoke đã xong. Sự tách đôi này vừa là feature vừa là footgun:

- **Feature.** 50 skill không tốn 50 × độ-dài-body token mỗi turn. Chỉ description tốn. Đó là lý do thư viện skill lớn khả thi.
- **Drift footgun.** Một description hứa X nhưng body làm Y khiến Claude invoke sai bối cảnh — và phát hiện mismatch quá muộn.
- **Supply-chain risk.** Một skill share từ repo ngoài có thể cặp description lành ("code formatter") với body độc (exfiltrate secret). Claude tin description để route, rồi execute body trong session của bạn. Đây là lý do Claude Code yêu cầu approval rõ ràng cho import từ ngoài.

> **Description là hợp đồng routing. Body là hợp đồng execution. Chúng có thể lệch nhau — đó vừa là feature vừa là lỗ hổng. Khi review skill của bên thứ ba, hãy đọc *body*, đừng chỉ đọc description.**

### Hai kiểu trigger: explicit vs implicit

Nếu description quyết định *có* invoke hay không, câu hỏi tiếp theo là *ai* quyết định *khi nào* — user, hay Claude. Lựa chọn đó có tên, và chọn sai default là lý do phần lớn các setup "magic framework" lung lay sau lần compaction đầu.

Nếu bạn từng dùng một trong các framework Claude Code phổ biến — `get-shit-done`, `oh-my-claudecode`, `ccpm`, hay bất kỳ bundle kiểu *"N skill + M agent trong một repo"* — bạn đã cảm được cái ma thuật. Gõ gì đó, Claude chọn đúng skill, mọi chuyện diễn ra. Cảm giác như tool thực sự hiểu workflow của bạn.

Dừng lại, và hỏi câu khó chịu: **cái skill đó có thực sự hiểu *codebase của bạn* không, hay nó đang áp lời khuyên chung chung vào code của bạn và hy vọng trúng?**

Xét một tình huống cụ thể. Bạn cài một skill `/security-review` phổ biến. Nó chạy checklist chung: SQL injection, XSS, CORS, secret trong code, auth. Hữu ích. Nhưng:

- Bạn viết một middleware tùy biến intercept mọi request và yêu cầu `id` không rỗng trên mọi `DELETE`. Skill chung chưa bao giờ nghe tới.
- Project bạn có quy ước mọi HTTP ra ngoài phải đi qua wrapper `httpclient.Do` duy nhất có retry và tracing. Skill chung mù về chuyện đó.
- Bạn enforce rằng package `internal/` không được import từ `cmd/`. Lại vô hình với framework không biết repo bạn.

Skill chung flag những vấn đề sách giáo khoa. Skill tùy biến theo codebase của bạn flag *những vấn đề của bạn* — loại mà review thực sự cần bắt. Magic là thứ giúp bạn khởi động. Customization là thứ biến Claude Code từ tool 2× thành tool 10×.

Và toàn bộ cảm giác "magic" xoay quanh đúng một cơ chế: **cách các framework đó trigger skill và agent.** Có hai cách:

1. **Explicit** — user gõ `/skill-name`, hoặc parent agent gọi tool `Agent` với subagent type cụ thể. Deterministic. User (hoặc parent model) nắm thời điểm.
2. **Implicit** — Claude đọc description mỗi turn, match với prompt hiện tại của user, và tự quyết định có invoke không. Đây là chế độ "nó đọc được suy nghĩ của tôi". Cũng là chế độ mà phần nhiều hành vi *trông* như thông minh thực ra là description engineering cẩn thận.

Claude Code mở các knob rõ ràng cho cả hai trong skill frontmatter:

| Frontmatter                      | Kiểu invoke                                        | Dùng cho                                                                                                                                |
| :------------------------------- | :------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------- |
| (default)                        | Cả `/command` và auto-trigger                      | Phần lớn skill                                                                                                                          |
| `disable-model-invocation: true` | Chỉ `/command`. Claude không thể auto-trigger.     | Mọi thứ có side effect: `/commit`, `/deploy`, `/send-slack-message`. Bạn không muốn Claude quyết *khi nào* deploy theo cảm tính.        |
| `user-invocable: false`          | Chỉ Claude auto-trigger. Ẩn khỏi menu `/`.         | Kiến thức nền (`legacy-system-context`, `api-conventions`) — không actionable như command, nhưng Claude nên kéo vào khi liên quan.      |
| `paths: src/billing/**`          | Claude auto-trigger chỉ khi đụng file khớp         | Skill đặc thù domain, không nên cạnh tranh attention ngoài khu vực của nó                                                               |

**Vì sao điều này quan trọng:**

- **Implicit trigger là nơi "magic framework" sống.** Khi một setup phổ biến có vẻ đọc được suy nghĩ, đó không phải đọc suy nghĩ — có ai đó đã engineer description, hint `when_to_use`, cách viết CLAUDE.md, và path scope để model auto-invoke đáng tin trên đúng prompt. Đó là công việc thật, và dễ fail. Một description phải *thắng cuộc cạnh tranh attention* với mọi skill khác trong budget.
- **Explicit trigger là cần gạt an toàn, đặc biệt với model yếu hơn.** Auto-trigger giả định model đủ thông minh để match prompt với description cho đúng. Trên model nhỏ / rẻ / nhanh hơn, auto-trigger lỗi nhiều hơn. Ép `/command` bỏ phần đoán.
- **Hành động có side effect gần như luôn nên explicit.** Nếu một skill deploy, gửi message, đổi shared state, hay tiêu tiền, bạn không muốn "Claude quyết định" trong post-mortem. `disable-model-invocation: true` là bảo hiểm rẻ.
- **Bơm kiến thức thuần thì implicit ổn.** Một skill nói *"khi đụng code billing, đây là các invariant"* là vật liệu auto-trigger lý tưởng — scope bằng `paths:`, giấu khỏi menu bằng `user-invocable: false`.

### Decision tree — "kiến thức này sống ở đâu?"

Trigger trả lời *khi nào* fire. Placement trả lời câu trước đó — *primitive nào* một mẩu kiến thức thuộc về. Cái đó xứng đáng có một cây. Bắt đầu từ trên; match đầu tiên thắng.

```
Có thể suy ra từ `git log` hoặc đọc code không?
  └── CÓ → không lưu gì cả. Đừng làm bẩn context.

Có phải MUST / MUST NOT luôn đúng, bất kể task?
  ├── Kiểm tra được bằng máy (grep, lint)? → rule + hook
  └── Chỉ enforce bằng con người?           → rule

Có phải workflow "khi user nói X, làm Y₁ → Y₂ → Y₃"?
  └── CÓ → skill

Để trả lời cần đọc >2000 dòng source?
  └── CÓ → subagent (cách ly khỏi context chính)

Có phải fact về user hoặc project, thay đổi theo thời gian?
  └── CÓ → memory

Có phải style / tone / triết lý phủ toàn repo?
  └── CÓ → CLAUDE.md
```

Nếu không match gì, nhiều khả năng bạn không cần persist. Cây là kỷ luật tối thiểu — mọi thứ trong repo này đều là nỗ lực biến nó thành phản xạ.

---

## 6. Sắp tới — và repo này KHÔNG phải gì

### Sắp có

- [ ] `docs/anti-patterns.md` — 6 kiểu đặt sai chỗ phổ biến nhất
- [ ] `docs/lifecycle.md` — khi nào nâng memory → rule, khi nào hạ CLAUDE.md → rules/
- [ ] `examples/` — một ví dụ tối giản, đã sanitize cho mỗi primitive
- [ ] Case study — một flow thật (viết test) chia ra 5 layer, kèm reasoning

Welcome issue và PR. Nếu bạn không đồng ý với một placement, mở issue — điểm chính là làm hiện lên các tradeoff.

### Ngoài phạm vi

- Không phải list "awesome-claude". Thứ đó có rồi.
- Không phải tutorial cài Claude Code. Xem [docs chính thức](https://docs.claude.com/en/docs/claude-code).
- Không phải tip prompt engineering. Ngoài scope.
- Không ý kiến về lựa chọn model. Quá biến động.

### Kêu gọi đóng góp — một dashboard context sống

Thesis trung tâm ở đây là **hiểu Claude đang biết gì ngay lúc này là một kỹ năng, và công cụ để nhìn thấy điều đó đang bị đầu tư thiếu.** `/context` là view tốt nhất hiện có, và nó là một đống text. Có một khoảng trống có thể xây giữa nó và một thứ thực sự hữu ích — session JSONL đã chứa ~80% dữ liệu cần để parse, gán chi phí, và visualize mọi lần context load theo thời gian thực. RFC đầy đủ, build plan từng bước (MVP → TUI → dashboard → team analytics), câu hỏi design mở, và cách tham gia: [`docs/dashboard-rfc.md`](./docs/dashboard-rfc.md).

---

## License

MIT — xem [LICENSE](./LICENSE).
