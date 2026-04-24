# Claude Code, đúng cách

*[English](./README.md) · **Tiếng Việt***

Trước khi bắt đầu, một điểm về cách các chatbot AI hay agentic tool thực sự chạy bên dưới (miễn là chúng vẫn dựa trên LLM) — nếu chưa thấm, toàn bộ bài viết sẽ không có chỗ bám:

Khi bạn chat với một chatbot hay một agentic tool, ban đầu bạn phải nêu vấn đề đang gặp, thứ muốn làm, giới thiệu các thứ... Theo kinh nghiệm dùng của tôi, thường phải qua vài turn qua lại thì câu trả lời của model mới thực sự bám đúng yêu cầu — coi đây là một quan sát kinh nghiệm, không phải mốc cố định nằm trong cơ chế. Để model "hiểu" được vấn đề qua từng turn thì bên dưới thực sự trông như sau: để turn thứ N vẫn "nhớ" được những gì turn 1→N-1 đã nói, client phải gửi lại *toàn bộ hội thoại turn 1→N-1 lên cho model ở mỗi turn, để nó tự suy nghĩ lại từ đầu* — nghe ảo không. Và đây là điểm quan trọng nhất về cách LLM hoạt động: *nó stateless*. Hồi LLM mới ra, tôi cũng háo hức clone một model về chạy trên máy — turn 1 giới thiệu *"tôi tên là…"*, turn 2 hỏi lại *"mày biết tao là ai không"*, bạn biết câu trả lời rồi đấy. Xong hí hửng đi tìm hiểu thì mới vỡ ra: phải gửi cả history cũ lên thì nó mới biết. Và về cơ bản, với giới hạn hiện tại, tổng lịch sử có thể gửi lên tối đa khoảng 1M token (một **token** xấp xỉ 3–4 ký tự tiếng Anh — 1M token tương đương cỡ vài trăm nghìn dòng code) — đây là lý do vì sao cần tổ chức cơ cấu đó phù hợp để phục vụ các bài toán to.

> **Về token.** Token không phải là ký tự, cũng không hẳn là từ. Nó là đơn vị văn bản mà model dùng để đọc và xử lý ngôn ngữ, và số token phụ thuộc vào tokenizer của từng model. Lấy câu *"tôi không muốn đọc bài này nữa"* làm ví dụ. Nếu cứ tách theo dấu cách cho đơn giản, bạn sẽ có 7 "từ": `tôi` | `không` | `muốn` | `đọc` | `bài` | `này` | `nữa`. Nhưng tokenizer thật không tách theo dấu cách — nó dùng subword, và một tokenizer phổ biến có thể cắt câu trên đại khái thành `tôi` | ` không` | ` mu` | `ốn` | ` đ` | `ọc` | ` bài` | ` này` | ` n` | `ữa` — khoảng 10 token cho một câu rất ngắn. Câu tiếng Anh tương đương *"I don't want to read this anymore"* thường chỉ tốn ~8 token.

Thấm rồi thì câu hỏi *"mọi thứ phải gửi lại mỗi turn — vậy đưa gì vào, không đưa gì vào?"* tự nó xuất hiện. Bài viết là câu trả lời cho câu hỏi đó.

Cơ chế này đúng cho mọi chatbot hay agentic tool — miễn là nó chạy trên một LLM. Bài viết này focus vào một agentic đang phổ biến: **Claude Code**. Nhưng bạn hoàn toàn có thể áp dụng logic tương tự cho các agentic khác — cách thiết lập, cách dùng, và cách tổ chức context về bản chất đều giống nhau.

Phần lớn chúng ta dùng Claude theo cách rất tự nhiên: nêu vấn đề, rồi để nó tự tìm đường giải quyết. Điều đó không sai. Claude được tạo ra để xử lý những bài toán mở, mơ hồ, thiếu cấu trúc — và phần lớn thời gian nó làm khá tốt.

Chính vì nó đủ thông minh, ta dễ hình thành thói quen phó thác.

Có bug thì quăng bug vào. Có feature thì ném spec vào. Có repo lạ thì bảo nó tự đọc rồi hiểu. Xong trao đổi vài turn để cùng làm rõ yêu cầu. Khi nó làm được điều gì đó khéo léo, ta càng tin rằng chỉ cần mô tả vấn đề đủ hay, phần còn lại model sẽ lo.

Cách dùng đó ổn — cho tới khi bạn cần sự ổn định.

Khi bạn muốn một tình huống mà nó đã từng xử lý tốt sẽ tiếp tục được xử lý tốt. Khi bạn muốn cùng một loại task hôm nay, tuần sau, hay tháng sau vẫn cho ra chất lượng tương đương. Khi bạn muốn kinh nghiệm từ những phiên trước được giữ lại như năng lực có thể tái sử dụng, thay vì biến mất rồi được phát minh lại ở session kế tiếp.

Lúc đó, giới hạn bắt đầu lộ ra.

Bạn nhận ra mình chưa xây một hệ thống đáng tin cậy. Bạn chỉ đang tận dụng trí thông minh của model trong từng khoảnh khắc. Trí thông minh (cơ học mà nói: *input → model inference → output*) tự nó là tốt — nhưng nếu bạn dùng nó để *viết chương trình*, trí thông minh đó phải ổn định, có cấu trúc, tuân theo quy chuẩn chung cho tất cả module trong hệ thống, đi theo convention. Vì viết code thì không thoát khỏi việc đó — tôi chưa thấy chương trình nào rốt cuộc không quy về những quy chuẩn code chung, nghiệp vụ rõ ràng, ràng buộc cụ thể. Còn nếu bạn chỉ muốn thả hồn để research hay chat, thì không cần.

Repo này tồn tại vì một lý do duy nhất: giúp bạn trao cho Claude đủ cấu trúc bền vững để phần thông minh, biết khám phá của model có thể đáp xuống một nơi có thể dự đoán được, thay vì hết phiên này đến phiên khác lại phải tự suy luận lại cùng những sự thật cũ về codebase của bạn.

Sự chuyển dịch mà nó hướng tới rất đơn giản: từ việc dùng một công cụ agentic như người dùng bình thường — âm thầm trầm trồ mỗi khi nó làm được điều gì đó khéo léo — sang dùng nó như một developer hiểu rõ cơ chế bên trong và chủ động điều khiển chúng.

Từ điểm đó, việc chọn model phù hợp cho từng tác vụ trở thành một núm chỉnh có chủ đích. Bạn có thể hạ xuống model rẻ hơn, yếu hơn mà không lo mọi thứ trật đường ray — bởi đường ray nằm ở cấu trúc bạn xây, chứ không nằm ở trí thông minh của model.

### Ai nên đọc

- **Người dùng Claude Code đã đủ giờ bay để nhận ra ít nhất một trong bốn triệu chứng ở [Mục 1](#1-delegate-dont-dictate-âm-thầm-đổ-vỡ-trong-session-dài)** — session anxiety, nhồi nhét, mê tín workflow, nghĩa địa `CLAUDE.md`. Nếu chưa cái nào đánh trúng, bookmark lại và quay lại sau vài tuần dùng thật. Hướng dẫn sẽ không đáp xuống trước khi cơn đau xuất hiện.
- **Developer muốn hiểu *tại sao* session dài đổ vỡ, không chỉ sưu tập mẹo.** Lập luận dựa trên cơ chế — context window, compaction, attention dilution — và kỳ vọng bạn đọc như một kỹ sư, không phải người đi theo công thức.
- **Người sẵn sàng đọc một thư mục `.claude/` như đọc source code.** Đây không phải một framework để cài. Mục 6 lập luận rằng thứ thật sự giao được là khả năng mở bất kỳ setup Claude Code nào — của bạn hay của người khác — và thấy cái gì đang thực sự diễn ra bên dưới.
- **Người đã thử nhiều framework và không còn biết cái nào thực sự work.** Nếu bạn đã gom skill, command, agent từ nửa tá repo và cả stack giờ giống một hộp đen thỉnh thoảng làm đúng, hướng dẫn này dành cho bạn. Mục tiêu không phải thêm một framework nữa trên cùng — mà là vốn từ cơ học để mở nắp capo những cái bạn đã có và biết bộ phận nào đang làm việc thật.

Đây **không** phải điểm khởi đầu đúng nếu bạn hoàn toàn mới với Claude Code (bạn cần cơn đau trước), hoặc nếu bạn đến để săn một framework adopt-sẵn (hướng dẫn cố tình không ship cái nào — nó dạy bạn chấm điểm những cái đã tồn tại).

> Đúc kết từ trải nghiệm hằng ngày trên một Go monorepo cỡ vừa. YMMV — coi đây là điểm khởi đầu, đừng coi là giáo điều.

### Mục lục

1. [**"Delegate, don't dictate" âm thầm đổ vỡ trong session dài**](#1-delegate-dont-dictate-âm-thầm-đổ-vỡ-trong-session-dài) — failure mode mà hướng dẫn này tồn tại để sửa: cách `/compact` và session mới đều bào mòn context bạn đã cẩn thận đưa vào, cùng bốn triệu chứng mà bất kỳ ai dùng Claude Code lâu năm đều nhận ra.
2. [**Nhìn xem Claude đang thực sự mang theo cái gì**](#2-nhìn-xem-claude-đang-thực-sự-mang-theo-cái-gì) — `/memory` và `/context`, hai câu lệnh chẩn đoán biến *"theo cảm giác"* thành thứ đo được, cộng thêm luận điểm: *sự không nhất quán của Claude Code thường là vấn đề context-architecture, không phải vấn đề model-intelligence.*
3. [**Vì sao model to hơn không cứu được bạn**](#3-vì-sao-model-to-hơn-không-cứu-được-bạn--ba-fact-về-llm-runtime) — ba fact về LLM runtime (window hữu hạn, gửi lại stateless, self-attention dilution) giải thích vì sao vấn đề scale theo tham vọng của bạn, chứ không theo model.
4. [**Các primitive**](#4-các-primitive--chỗ-bền-vững-nằm-ngoài-compact) — sáu chỗ bền vững để đặt kiến thức mà `/compact` không viết lại được: CLAUDE.md, rule, skill, hook, memory, subagent.
5. [**Cơ chế placement**](#5-placement--ba-trục-hai-kiểu-trigger-một-decision-tree) — ba trục chi phí, ranh giới tin cậy description-vs-body, trigger explicit-vs-implicit, cách đặt nội dung rule, skill-như-template, và một decision tree cho việc mỗi mẩu kiến thức sống ở đâu.
6. [**Đọc bất kỳ framework nào, tự mình**](#6-đọc-bất-kỳ-framework-nào-tự-mình) — cơ bắp mà hướng dẫn này xây: giờ bạn có thể mở bất kỳ thư mục `.claude/` nào và thấy cơ chế đang làm việc thật, không phải magic.
7. [**Sắp tới**](#7-sắp-tới--và-repo-này-không-phải-gì) — docs sắp ra, ranh giới scope, và một lời kêu gọi cộng tác viên cho dashboard context sống.

Trước khi bàn *cách* tổ chức, nên thành thật một chút về chuyện gì sẽ hỏng nếu không tổ chức — vì đó là tình trạng của đa số session Claude Code hiện nay.

---

## 1. "Delegate, don't dictate" âm thầm đổ vỡ trong session dài

Cách đa số người dùng Claude Code hiện nay là **giao phó ý định (delegating intention)** — và chính docs của Anthropic tích cực khuyến khích điều này. Lời khuyên của họ rất rõ: *["Delegate, don't dictate. Think of delegating to a capable colleague. Give context and direction, then trust Claude to figure out the details."](https://code.claude.com/docs/en/how-claude-code-works#delegate-dont-dictate)* Và cách họ mô tả nó hoạt động là: *"When you give Claude a task, it works through three phases in a loop: gather context → take action → verify results."*

Quay lại với mục đích của repo — *"Repo này tồn tại vì một lý do duy nhất: giúp bạn trao cho Claude đủ cấu trúc bền vững để phần thông minh, biết khám phá của model có thể đáp xuống một nơi có thể dự đoán được, thay vì hết phiên này đến phiên khác lại phải tự suy luận lại cùng những sự thật cũ về codebase của bạn."* Ánh xạ vào đây, nó sẽ là làm cho phần việc *gather context* trở nên **stable, stateful** hơn với các bài toán, vấn đề tương tự trong quá khứ, chứ không phụ thuộc vào cảm xúc của người prompt. Hay nói một cách cụ thể hơn:

*"Give context and direction"* là một **lời hứa có trạng thái (stateful promise)** — context và direction bạn đưa ở turn 1 phải còn hiệu lực khi Claude ra tay ở turn 47. Lời hứa đó không sống sót qua session dài, vì ngay khi cuộc trò chuyện to đến mức căng context window, Claude Code không từ chối tiếp tục — nó kích hoạt **compaction** (chạy command `/compact`), và âm thầm viết lại những turn cũ thành một bản tóm tắt để session còn chạy được.

Nói nhanh về `/compact`, vì mọi thứ bên dưới đều dựa vào nó. Khi một session Claude Code tiến sát trần context (1M token với Opus 4.7), client sẽ chạy **compaction**: các turn cũ — prompt cũ của bạn, mọi tool result, mọi file đã đọc, mọi reply Claude viết — bị **model tự tóm tắt thành một khối có cấu trúc ngắn**. Nôm na, vài trăm nghìn token lịch sử nguyên văn sụp xuống còn vài chục nghìn token tóm tắt (khoảng ~800k → ~100–200k tùy nội dung). Phần nguyên văn biến mất; chỉ bản tóm tắt sống tiếp sang turn sau. Đây là thứ giữ cho session dài không chết quá trần window, và cũng là nơi phần lớn context bạn cẩn thận đưa vào *chết âm thầm*.

Sau một lần `/compact`, direction ban đầu của bạn không còn là instruction nguyên văn nữa — nó là một vài câu nằm trong bản tóm tắt do model tự viết. Sau ba chục lần `/compact` trong một tuần, nó là một đống tóm-tắt-của-tóm-tắt có thể không còn khớp với nhau. Bạn không thể đòi Claude, sau lần compaction thứ 30, nhớ lại toàn bộ ý đồ bạn đặt ra trước lần đầu tiên. Docs im lặng về chuyện này.

Lối thoát thực sự là *ghi lại những thứ cần thiết vào đâu đó bền vững, rồi đưa chúng trở lại vào prompt sau mỗi lần `/compact`* — với cách đó, session dù đã compact N lần vẫn chạy theo ý bạn. Và khi session đi chệch tới mức không cứu được, lựa chọn cuối là đập đi xây lại — mở session mới.

Câu hỏi thật tình phải đặt ra là: *bạn có thực sự biết Claude đang chú ý vào cái gì ngay lúc này không?* Vì nếu bạn có ghi lại rồi re-inject hoặc mở session mới, nhưng không biết ngoài ra Claude còn nhớ thêm gì nữa hay không, thì điều đó vẫn chưa hoàn hảo. Nếu thành thật, câu trả lời thường là "theo cảm giác". Và cảm giác mơ hồ đó biểu hiện qua vài triệu chứng rất cụ thể mà bất kỳ ai dùng Claude Code đã lâu đều nhận ra:

- **Session anxiety — nỗi lo session.** Bạn làm với Claude được 90 phút. Mọi thứ đang trôi. Bạn không dám đóng session. Không phải vì biết Claude sẽ quên cái gì cụ thể — mà vì *bạn không biết nó đang nhớ cái gì, chấm hết.* Với đầu óc kỹ sư, đó là loại dependency tệ nhất: **không biết cái mình không biết**.
- **Nhồi nhét (cramming).** Claude vừa fix một edge case. Bạn lập tức cố fix hết mọi edge case tương tự trong cùng session — hoặc hí húi viết docs, thầm cầu nguyện *"lần sau Claude sẽ đủ khôn để nhớ"*. Bạn làm vậy không vì đó là workflow đúng. Bạn làm vì không tin session mới sẽ rơi vào đúng trạng thái attention này.
- **Mê tín workflow (workflow superstition).** Claude vừa chạy xong setup integration-test 5 bước: *spin up env → khởi động service phụ thuộc → override local config → chạy test case → in báo cáo dễ đọc*. Mọi thứ khớp. Bạn ghi nhận, với chút khó chịu, rằng bạn rất không muốn làm lại từ đầu — dù về mặt logic, toàn bộ state đó nằm trong file và shell command, không phải trong đầu Claude.
- **Nghĩa địa `CLAUDE.md` (the `CLAUDE.md` graveyard).** Sau mỗi lần chỉnh đau đớn, bạn nhét thêm một bullet vào `CLAUDE.md`. Sáu tháng sau, file dài 400 dòng và bạn không còn chắc bullet nào đang thực sự kích hoạt, bullet nào mâu thuẫn với bullet nào, hay bullet nào đã hết đúng từ hai sprint trước.

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

- **Skill description là khối luôn-được-load duy nhất bị `/compact` âm thầm bỏ.** `/command` invocation vẫn chạy sau compaction (harness load skill body trực tiếp từ đĩa bất kể ra sao), nhưng auto-trigger *implicit* của Claude không còn thấy bất cứ skill nào chưa từng được invoke — lý do vì sao các framework auto-triggering "magic" rõ ràng đần đi sau session dài.
- **Path-scoped rule và nested `CLAUDE.md` là những mục dễ gãy nhất trong `/memory`.** Chúng nằm trong message history, nên compaction xóa sạch, và chỉ quay lại khi Claude tình cờ đọc lại một file khớp path.

Bảng lifecycle đầy đủ — mục nào sống qua `/compact`, mục nào được prefix-cached, mục nào reload theo yêu cầu — xem [`docs/llm-mechanics/memory-entry-lifecycle.md`](./docs/llm-mechanics/memory-entry-lifecycle.md).

### Cái nhìn budget: `/context`

`/memory` cho biết *file nào* đã load. Nó không cho biết *tốn bao nhiêu*. Cho việc đó, có câu lệnh thứ hai, và có thể nói là còn hữu ích hơn:

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

Một câu chuyện cảnh báo về việc *Memory files* có thể tệ đến đâu. Một người bạn từng chạy `/init` khi đang đứng trong một directory tình cờ chứa **100 project**. Claude ngoan ngoãn scan hết từng project và scaffold ra một `CLAUDE.md` quái vật cỡ **~1 triệu token** — về cơ bản là mọi project trong folder đó bị flatten thành context-priors. Từ đó trở đi, mọi session mới mở ra đều đã đầy ~95% window 1M-token *trước khi anh ta gõ một prompt*. Response chậm, đắt, và hơi loạn, và gần trọn một ngày anh ta không biết tại sao. Đó là chuyện xảy ra khi `CLAUDE.md` rơi vào sai cấp directory: nó không còn là file config nữa mà thành khoản thuế định kỳ mỗi turn, lớn hơn cả cuộc hội thoại thật. Luôn scaffold `/init` ở *project* root, đừng bao giờ ở folder-chứa-nhiều-project.

**Điều quan trọng nhất phải thấm trước mọi thứ khác: sự không nhất quán của Claude Code thường là vấn đề context-architecture, không phải vấn đề model-intelligence.** Nếu *Messages* chiếm ưu thế trong output `/context`, một lần `/compact` hoặc session mới sẽ fix. Nếu *System tools*, *Memory files*, hoặc *Skills* to không tương xứng, bạn có **vấn đề structural** — đổi session không cứu được — bạn phải cắt bớt một MCP server, xóa một rule lỗi thời, hoặc đánh dấu skill là `disable-model-invocation: true`. Dù kiểu gì, fix nằm trong file của bạn, không nằm trong một model "thông minh hơn".

#### Mấy con số token này có thật không? Kiểm chéo ba hướng

Hoài nghi hợp lý: liệu `/context` có đang hiển thị số "minh họa" không phản ánh cái thực sự gửi đến Anthropic? Ba nguồn độc lập nói không — mọi token ở đó là token thật trong một API request thật:

1. **Docs Anthropic** nói thẳng: *["Skill descriptions are loaded into context so Claude knows what's available."](https://code.claude.com/docs/en/skills)* Tương tự cho CLAUDE.md, rule, và auto-memory.
2. **Session log chứa nguyên văn.** Session JSONL của bạn tại `~/.claude/projects/<project>/<session-id>.jsonl` chứa một `skill_listing` attachment ở turn đầu, với **toàn bộ text** của mọi skill description nối lại. Chia độ dài ký tự cho ~4 thì khớp với con số "Skills" mà `/context` báo. Các attachment tương tự (`nested_memory`, `deferred_tools_delta`) ghi lại nội dung của mọi rule, CLAUDE.md, và MCP tool list đã load.
3. **Accounting API khớp.** Giá trị `cache_creation_input_tokens` của assistant message đầu tiên bằng tổng của toàn bộ category startup trong `/context` cộng một phần framing overhead nhỏ. Nếu một category là giả, con số này sẽ thấp hơn đúng bằng kích thước category đó.

**Vì sao quan trọng (và nó tốn bạn bao nhiêu):**

- Mỗi skill, mỗi rule, mỗi MCP tool đã load đều tốn token **ở mỗi turn**, không chỉ khi được invoke. Cache hit làm turn lặp rẻ (0.1× giá base input), nhưng bất kỳ cache miss nào — idle quá 5 phút TTL mặc định, edit `CLAUDE.md`, thêm skill mới — đều viết lại toàn bộ prefix với giá 1.25× base.
- Một framework kiểu *"50 skill trong một repo"* × ~100 token mỗi description = **~5k token mỗi turn, mãi mãi**, dù bạn có dùng skill đó hay không. Đây không phải chi phí cài đặt một lần; đây là thuế định kỳ trên mọi session.
- `disable-model-invocation: true` *thực sự* đưa một skill ra khỏi khoản thuế đó. Description của nó không bao giờ được thêm vào startup prompt, nên tốn 0 token ở những turn bạn không invoke bằng `/name`.

> **Deep dive:** [`docs/llm-mechanics/per-turn-cost-math.md`](./docs/llm-mechanics/per-turn-cost-math.md) — bảng giá đầy đủ. Ví dụ thực tế cho biết một lần edit `CLAUDE.md`, một lần pha cà phê 10 phút, hay một MCP server phì đại thực sự tốn bao nhiêu cho 100-turn session, với số Opus 4.7. Có phân tích break-even cho tier cache 1 giờ.

Mối quan hệ trung thực giữa hai câu lệnh:

- **`/context`** là khung chẩn đoán toàn diện — mọi category nội dung hiện đang (hoặc sẽ) vào context, có chi phí token, drill theo item. Tất cả những gì `/memory` hiện là tập con chặt của những gì `/context` hiện, *cộng* token, *cộng* skill, agent, MCP tool, và phần accounting Messages / Free-space / Autocompact-buffer.
- **`/memory`** là editor cho memory file. Nó liệt kê cùng bộ `CLAUDE.md`, rule, auto-memory file, nhưng nhiệm vụ của nó là cho bạn *sửa* — chọn một file thì mở trong editor, hoặc bật/tắt auto-memory. Để chẩn đoán, `/context` mạnh hơn hẳn; `/memory` thắng khi bạn muốn fix ngay một file vừa phát hiện.

`/memory` và `/context` cho biết *cái gì* Claude đang mang, và *tốn gì*. Chúng không thể cho biết *tại sao* phải cắt giảm, hay tại sao context window to hơn không phải là giải pháp thay thế. Cả hai câu hỏi đó có chung một câu trả lời — ba fact về cách LLM thực sự chạy tại inference time.

---

## 3. Vì sao model to hơn không cứu được bạn — ba fact về LLM runtime

Không cần hiểu sâu ML để đặt kiến thức đúng chỗ trong Claude Code. Bạn chỉ cần ba fact, ở dạng vừa đủ. Mọi quyết định placement ở phần còn lại của hướng dẫn này đều bắt nguồn từ ba fact này.

### Fact 1 — Context window có giới hạn, và được dùng chung

Opus 4.7 có giới hạn 1M token. Nghe khổng lồ (với người đã đọc [session LLM](./docs/llm-mechanics/self-attention.md) rồi thì thấy nó khổng lồ, chứ so 1M với dung lượng bộ nhớ, RAM... thì muỗi — nó mà đạt đến mức hàng TB như bộ nhớ thì giờ chúng ta đi làm nông dân hết rồi); thật ra không, vì budget được **dùng chung**, không dành riêng cho câu hỏi của bạn. Mọi thứ bên dưới tranh cùng 1M đó *ở mỗi turn*:

- System prompt của chính Claude Code
- Mọi tool schema — built-in, MCP, agent definition
- Mọi `CLAUDE.md`, mọi rule luôn-load, mọi skill description, `MEMORY.md`
- Toàn bộ hội thoại trước, mọi tool result, mọi file Claude đã đọc

Một lần đọc file source 2000 dòng có thể ăn 30–50k token. Mười lần đọc như vậy trong một session debug dài và bạn đã đốt 5% window trước khi làm việc thật. "1M token" là trần, không phải workspace — workspace dùng được co lại mỗi phút trong session.

### Fact 2 — Mỗi request gửi lại toàn bộ hội thoại

Window hữu hạn sẽ dễ chịu nếu model giữ được state giữa các turn. Nó không. Inference LLM là **stateless** — client gửi lại toàn bộ hội thoại trước, mọi `CLAUDE.md` đã load, mọi rule, mọi skill description, mọi tool definition, **mỗi turn**, từ turn 1.

Hệ quả thực tế: thêm 500 dòng vào `CLAUDE.md` không phải chi phí một lần. Nó là thuế mỗi turn, nhân với mọi turn của mọi session tương lai. Prompt caching làm nhẹ tác động (0.1× cho prefix khớp trong TTL 5 phút), nhưng caching chỉ giúp nếu bạn không invalidate prefix — và edit `CLAUDE.md` giữa session invalidate mọi thứ phía sau.

### Fact 3 — Context to hơn không có nghĩa là focus tốt hơn

Trả phí mỗi turn cũng sẽ dễ chịu nếu mỗi token bạn trả tiền đều làm việc đều nhau. Nó không. Self-attention — cơ chế model dùng để quyết định token nào quan trọng — gán cho mỗi token một trọng số, và **các trọng số phải cộng lại thành một tổng cố định**. Mỗi token không liên quan thêm vào làm giảm trọng số dành cho những token liên quan. Nhiều context hơn làm signal *mỏng đi*, không sắc hơn. Kết hợp với hiệu ứng "lost in the middle" đã được đo đạc thực nghiệm (thông tin bị vùi giữa prefix dài và suffix dài bị trọng số thấp hơn), một prompt dài gấp đôi có thể *kém hiệu quả hơn* một prompt ngắn bằng nửa, nếu phần thêm vào là noise.

> **Deep dive:** [`docs/llm-mechanics/self-attention.md`](./docs/llm-mechanics/self-attention.md) minh họa vì sao đây là vấn đề *cơ học*, không phải bug mà một window to hơn sẽ sửa. [`docs/llm-mechanics/why-context-matters.md`](./docs/llm-mechanics/why-context-matters.md) mở rộng cả ba fact với ví dụ cụ thể về cái giá mỗi fact bắt bạn trả trong thực tế.

### Quỹ đạo có đáy

Ba fact này không được fix bởi một window to hơn. Chúng *cộng dồn* với nó. Model sẽ tiếp tục thông minh lên; context window tiếp tục to ra — 8k → 200k → 1M → cái tiếp theo. Các thắng lợi kỹ thuật như prompt caching, path-scoped rule, compaction, subagent isolation đều đẩy trần lên. Nhưng chúng không đổi *hình dạng* của trần: **vấn đề scale theo tham vọng của bạn, không theo model.** Ngày window 10M-token đến, bạn sẽ muốn nhét 20M token context project vào đó.

### Reasoning không phải là durable knowledge

Thêm một fact nữa, lần này là quan sát thực tế. Tôi từng apply một config Terraform và fail. Không cần hand-holding, Claude đoán URL có thể sai, đi tìm source của provider, không tìm được local (provider được đặt tên trái quy ước), web-search tên gần nhất, tìm thấy GitHub repo, đọc source, và phát hiện một dấu `/` thừa không nên có ở cuối. Mind-blowing. Và tất nhiên sau đó tôi không dám đóng session — chắc chắn cái expertise đó đang sống *đâu đó*. Đúng là sau N turn làm việc không liên quan, một vấn đề tương tự xuất hiện và Claude phải suy luận lại một nửa từ đầu.

Một model đủ thông minh để *suy ra* câu trả lời đúng một lần vẫn sẽ suy ra lại, *không hoàn hảo*, mỗi khi context của nó drift hay reset. Trong session dài, attention rot dần dần chôn vùi câu trả lời. Trong session mới, nó chưa bao giờ có ở đó.

> **Deep dive:** [`docs/llm-mechanics/delegation-ceiling.md`](./docs/llm-mechanics/delegation-ceiling.md) cho câu chuyện đầy đủ và vì sao chỉ model tốt hơn không thu hẹp được khoảng cách.

### Tóm lại: cơ chế, không phải độ thông minh

Sau ba mục, đến lúc chốt lại trước khi đi tiếp. Vì nếu đến đây bạn vẫn chưa thấy *tổ chức lại cách dùng Claude là quan trọng và cần thiết*, thì không nên đi tiếp — hoặc tôi sai (không có chuyện đó), hoặc bạn sai.

Nếu có một câu duy nhất tổng kết ba mục đầu: vấn đề không nằm ở chỗ Claude thông minh đến đâu — nó nằm ở *cơ chế LLM chạy tại inference time*. Mục 1 cho bạn bốn triệu chứng của việc không biết Claude đang nhớ gì. Mục 2 cho bạn hai lệnh để đo chuyện đó. Mục 3 cho bạn ba fact nói vì sao bigger model hay bigger window không sửa được gốc. Cả ba gộp lại thay đổi **kỳ vọng** của bạn về việc Claude "nhớ" công việc: nó có một context window mọi thứ bạn load vào đều tranh nhau, nạp lại từ đầu ở mỗi turn, và bị tóm tắt âm thầm khi chạm ngưỡng giới hạn.

Trong điều kiện đó, việc *tổ chức cái gì cần nhớ, đặt ở đâu, nạp lại lúc nào* không còn là tối ưu chi tiết — nó là yêu cầu cơ bản để session dài không đổ vỡ.

Đến đây bạn có thể dừng đọc và tự đi research tiếp — bạn đã có câu hỏi rồi. Các giải pháp ở dưới chủ yếu là về mặt ý tưởng. Có một phần tôi tâm huyết: so sánh các cách làm việc mà các framework Claude Code thi nhau đẻ ra để giải quyết các vấn đề trên.

Một điều cần thành thật thêm: bài viết này ra đời để *tôi* tự trả lời vấn đề của *tôi* — một developer luôn thấy khó chịu khi dùng một giải pháp mà không hiểu cơ chế nó chạy. Nó không phải cho bạn. Sorry vì đã lừa bạn tới đây. Nhưng nếu bạn cũng thuộc type người này — muốn hiểu bản chất của cái mình dùng, rồi *dùng nó* (không phải viết lại toàn bộ từ đầu) — thì đi tiếp.

---

## 4. Các primitive — chỗ bền vững nằm ngoài `/compact`

Sáu primitive, mỗi cái trả lời một biến thể của cùng một câu hỏi: *đặt cái này vào đâu để nó sống qua `/compact`, và chỉ vào lại context ở đúng những turn cần?*

- **`CLAUDE.md` và rule** đóng gói kiến thức project bền vững và đưa vào context ở mỗi turn liên quan. Body là instruction; Claude đọc thụ động.
- **Skill và subagent** đóng gói kiến thức *thủ tục* — *"khi thấy X, làm Y₁ → Y₂ → Y₃"* — và trình cho model kèm metadata (`description`, `when_to_use`, `paths`) về lúc nào nên với tới. Body không vào context cho đến khi được invoke. Subagent còn isolate công việc của mình trong một context window mới, nên cuộc hội thoại chính không bị ô nhiễm.
- **Hook** thực thi số ít invariant mà không thể tin model tự giữ. Chúng chạy hoàn toàn bên ngoài model, nên miễn nhiễm với attention rot, context drift, hay một prompt khéo léo thuyết phục Claude đi ngược.
- **Memory file** lưu lại fact user-local xuyên session — những thứ về bạn hay project thay đổi theo thời gian nhưng không nên gõ lại mỗi session.

<details>
<summary>Mở ra nếu cần nhắc lại các tên này.</summary>

**`CLAUDE.md`** — một file markdown ở root repo (hoặc root subdirectory) mang **project context bền vững** Claude nên biết ở đầu mỗi session: build command, quy ước, ghi chú kiến trúc, quy tắc đặt tên, workflow thường dùng, những thứ bạn ngán phải giải thích lại. Claude đọc mỗi turn, nên đây cũng là chỗ đặt behavioral guideline ("ưu tiên edit thay vì tạo file mới", "đừng thêm comment trừ khi được yêu cầu"). Có thể scaffold file ban đầu bằng `/init` trong project. Một `CLAUDE.md` lỗi thời, mâu thuẫn, hay dài 400 dòng là nguồn gốc số một của lời than "Claude Code cứ làm sai". [Docs chính thức](https://docs.claude.com/en/docs/claude-code/memory).

**Rule** (`.claude/rules/*.md`) — file markdown dưới `.claude/rules/` mà Claude Code tự quét đệ quy và load với priority ngang `.claude/CLAUDE.md`. Dùng để tách invariant project ("MUST / MUST NOT") ra khỏi một `CLAUDE.md` phì đại. YAML `paths:` frontmatter scope rule vào pattern file cụ thể để nó chỉ load khi Claude đụng file khớp. Rule mức user ở `~/.claude/rules/`. [Docs chính thức](https://code.claude.com/docs/en/memory#organize-rules-with-claude/rules/).

**Skill** (`.claude/skills/<name>/SKILL.md`) — workflow được đặt tên và trigger-based mà Claude có thể invoke. Mỗi skill có description nói Claude khi nào dùng, cùng các bước procedural bên trong. Lý tưởng cho "khi user nói X, làm Y₁ → Y₂ → Y₃". [Docs chính thức](https://docs.claude.com/en/docs/claude-code/skills).

**Hook** (`.claude/hooks/*.sh` + `settings.json`) — shell script mà harness Claude Code chạy tự động ở các sự kiện lifecycle (pre-tool, post-tool, user-prompt-submit, v.v.). Exit code khác 0 có thể chặn action. Đây là layer *thực thi bằng máy* — thứ duy nhất chặn Claude vi phạm một rule *kể cả khi* rule đã nằm trong context. [Docs chính thức](https://docs.claude.com/en/docs/claude-code/hooks).

**Memory** (`~/.claude/projects/<project>/memory/*.md`) — local theo user, lưu xuyên session. Claude tự viết vào đây để nhớ fact về bạn, project, và preference của bạn. *Không* dành cho những thứ code hay `git log` trả lời được. [Docs chính thức](https://docs.claude.com/en/docs/claude-code/memory).

**Subagent** (invoke qua tool `Agent`) — một Claude instance mới với context window riêng, spawn ra để xử lý một task có phạm vi. Parent chỉ thấy summary cuối cùng. Dùng để cách ly việc đọc nặng (hàng nghìn dòng), chạy công việc song song độc lập, hoặc chạy một model rẻ hơn trên một job có phạm vi hẹp mà không sợ nó đi chệch đường ray — *chính* role scope là guardrail, điều đó làm cho việc downshift model an toàn. [Docs chính thức](https://docs.claude.com/en/docs/claude-code/sub-agents).

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

Đặt tên sáu primitive là nửa dễ. Nửa khó là biết *cái nào* khớp với một mẩu kiến thức cụ thể — và nhận ra rằng chọn sai không phải vấn đề phong cách, mà là một khoản thuế định kỳ trên mọi turn tương lai của mọi session tương lai. Phần còn lại của hướng dẫn là mental model cho việc đặt ở đâu.

---

## 5. Placement — ba trục, hai kiểu trigger, một decision tree

Các quyết định placement trở nên dễ hơn nhiều khi bạn nhận ra rằng các primitive này nằm trên **ba trục khác nhau**, không phải một.

### Ba trục chi phí

**Trục 1 — Thuế context mỗi turn.** Những cái này vào context Claude ở đầu mỗi cuộc trò chuyện, dù bạn có dùng hay không:

- `CLAUDE.md` — toàn bộ nội dung
- `.claude/rules/*.md` không có `paths:` frontmatter — toàn bộ nội dung
- **Description** của skill (YAML frontmatter) — chỉ metadata, nhưng load cho *mọi* skill để Claude biết khi nào trigger
- **Description** của subagent — tương tự
- `MEMORY.md` — 200 dòng / 25KB đầu

Phải tính ngân sách cho hợp lý. Một repo 50 skill × 20 dòng description = ~1000 dòng thuế trả trước, trước khi bạn gõ một chữ.

**Trục 2 — Bổ sung prompt theo yêu cầu.** Những cái này chỉ vào context khi Claude quyết định chúng liên quan:

- **Body** của skill (mọi thứ trong `SKILL.md` sau frontmatter) — load khi Claude invoke skill
- **Execution context** của subagent — isolate trong window mới; parent chỉ thấy summary cuối
- Rule có path-scope (`paths: src/api/**`) — load khi Claude đọc file khớp
- Memory topic file (mọi file khác ngoài `MEMORY.md`) — load theo yêu cầu
- `CLAUDE.md` nested trong subdirectory — load khi Claude đụng file trong đó

Đây là chỗ của nội dung nặng, đặc thù task. Được miễn khỏi budget "mỗi turn".

**Trục 3 — Hoàn toàn bên ngoài model.** Hook là loại khác hẳn. Chúng chạy trong harness Claude Code (client Node.js), không trong model:

- Không tốn context token
- Deterministic: exit code của shell chặn / cho qua; model không override được
- Không thể bị "quên" giữa hội thoại
- Bị giới hạn ở những gì một shell command thực sự có thể verify

Hook là layer duy nhất biến "MUST NOT" từ hy vọng thành thực thi. Mọi primitive khác đều phụ thuộc việc Claude đọc, hiểu, và chọn tuân theo.

### Ranh giới tin cậy trong skill và subagent

Hai trong ba trục sống *bên trong* skill và subagent cụ thể, vì các primitive đó có một tính chất tách đôi mà các primitive khác không có: nửa metadata sống ở Trục 1, nửa body sống ở Trục 2.

|                 | Khi nào Claude thấy         | Để làm gì                         |
| --------------- | --------------------------- | --------------------------------- |
| **Description** | Mỗi turn (Trục 1)           | Routing: *"có nên invoke không?"* |
| **Body**        | Chỉ sau khi invoke (Trục 2) | Execution: *"giờ làm gì?"*        |

Claude route chỉ dựa trên description. Body không có chỗ để "phản đối" cho đến khi quyết định invoke đã xong. Sự tách đôi này vừa là design feature vừa là footgun:

- **Feature.** 50 skill không tốn 50 × độ-dài-body token mỗi turn. Chỉ description của chúng tốn. Đó là lý do thư viện skill lớn khả thi.
- **Drift footgun.** Một description hứa X nhưng body làm Y khiến Claude invoke sai bối cảnh — và phát hiện mismatch quá muộn.
- **Supply-chain risk.** Một skill share từ repo ngoài có thể cặp description lành ("code formatter") với body độc (exfiltrate secret). Claude tin description để route, rồi execute body trong session của bạn. Đây là lý do Claude Code yêu cầu approval rõ ràng cho import từ ngoài.

> **Description là hợp đồng routing. Body là hợp đồng execution. Chúng có thể lệch nhau — đó vừa là feature vừa là lỗ hổng. Khi review một skill của bên thứ ba, hãy đọc *body*, đừng chỉ đọc description.**

### Hai kiểu trigger: explicit vs implicit

Nếu description quyết định *có* invoke hay không, câu hỏi tiếp theo là *ai* quyết định *khi nào* — user, hay Claude. Lựa chọn đó có tên, và chọn sai default là lý do phần lớn các setup "magic framework" lung lay sau lần compaction đầu.

Nếu bạn từng dùng một trong các framework Claude Code phổ biến — `get-shit-done`, `oh-my-claudecode`, `ccpm`, hay bất kỳ bundle kiểu *"N skill + M agent trong một repo"* — bạn đã cảm được cái magic. Gõ gì đó, Claude chọn đúng skill, mọi chuyện diễn ra. Cảm giác như tool thực sự hiểu workflow của bạn.

Dừng lại, và hỏi câu khó chịu: **cái skill đó có thực sự hiểu *codebase của bạn* không, hay nó đang áp lời khuyên chung chung vào code của bạn và hy vọng trúng?**

Xét một tình huống cụ thể. Bạn cài một skill `/security-review` phổ biến. Nó chạy checklist chung: SQL injection, XSS, CORS, secret trong code, auth. Hữu ích. Nhưng:

- Bạn viết một middleware tùy biến intercept mọi request và yêu cầu `id` không rỗng trên mọi `DELETE`. Skill chung chưa bao giờ nghe tới.
- Project bạn có quy ước mọi HTTP ra ngoài phải đi qua một wrapper `httpclient.Do` duy nhất có retry và tracing. Skill chung mù về chuyện đó.
- Bạn enforce rằng package `internal/` không được import từ `cmd/`. Lại vô hình với framework không biết repo bạn.

Skill chung flag những vấn đề sách giáo khoa. Skill tùy biến theo codebase của bạn flag *những vấn đề của bạn* — loại mà review thực sự cần bắt. Magic là thứ giúp bạn khởi động. Customization là thứ biến Claude Code từ tool 2× thành tool 10×.

Và toàn bộ cảm giác "magic" đó xoay quanh đúng một cơ chế: **cách các framework đó trigger skill và agent.** Có hai cách:

1. **Explicit** — user gõ `/skill-name`, hoặc parent agent gọi tool `Agent` với một subagent type cụ thể. Deterministic. User (hoặc parent model) nắm thời điểm.
2. **Implicit** — Claude đọc description mỗi turn, match với prompt hiện tại của user, và tự quyết định có invoke không. Đây là chế độ "nó biết được cái tôi muốn". Cũng là chế độ mà phần nhiều hành vi *trông* như intelligence thực ra là description engineering cẩn thận.

Claude Code mở các knob rõ ràng cho cả hai trong skill frontmatter:

| Frontmatter                      | Kiểu invoke                                        | Dùng cho                                                                                                                                |
| :------------------------------- | :------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------- |
| (default)                        | Cả `/command` và auto-trigger                      | Phần lớn skill                                                                                                                          |
| `disable-model-invocation: true` | Chỉ `/command`. Claude không thể auto-trigger.     | Mọi thứ có side effect: `/commit`, `/deploy`, `/send-slack-message`. Bạn không muốn Claude quyết *khi nào* deploy theo cảm tính.        |
| `user-invocable: false`          | Chỉ Claude auto-trigger. Ẩn khỏi menu `/`.         | Kiến thức nền (`legacy-system-context`, `api-conventions`) — không phải dạng command thực thi được, nhưng Claude nên kéo vào khi liên quan.      |
| `paths: src/billing/**`          | Claude auto-trigger chỉ khi đụng file khớp         | Skill đặc thù domain, không nên cạnh tranh attention ngoài khu vực của nó                                                               |

**Vì sao điều này quan trọng:**

- **Implicit trigger là nơi "magic framework" sống.** Khi một setup phổ biến có vẻ đọc được suy nghĩ của bạn, đó không phải đọc suy nghĩ — có ai đó đã engineer description, hint `when_to_use`, cách viết CLAUDE.md, và path scope để model auto-invoke đáng tin trên đúng prompt. Đó là công việc thật, và dễ fail. Một description phải thắng *cuộc cạnh tranh attention* với mọi skill khác trong budget.
- **Explicit trigger là cần gạt an toàn, đặc biệt với model yếu hơn.** Auto-trigger giả định model đủ thông minh để match một prompt với description cho đúng. Trên model nhỏ / rẻ / nhanh hơn, auto-trigger trượt nhiều hơn. Ép `/command` bỏ phần đoán.
- **Hành động có side effect gần như luôn nên explicit.** Nếu một skill deploy, gửi message, mutate shared state, hay tiêu tiền, bạn không muốn "Claude quyết định làm" xuất hiện trong post-mortem. `disable-model-invocation: true` là bảo hiểm rẻ.
- **Bơm kiến thức thuần thì implicit ổn.** Một skill nói *"khi đụng code billing, đây là các invariant"* là vật liệu auto-trigger lý tưởng — scope bằng `paths:`, giấu khỏi menu bằng `user-invocable: false`.

### Nội dung của một rule thực sự sống ở đâu

Quyết định rằng một mẩu kiến thức là một *rule* chưa cho biết text của rule đó nên đặt ở đâu. Cùng một MUST / MUST NOT có thể đặt theo ba cách khác nhau, mỗi cách có hình dạng coverage-vs-cost khác:

1. **Centralized** trong `.claude/rules/*.md` không có `paths:` — load trên **Trục 1** mỗi turn. Đắt, nhưng **luôn được enforce**, dù user gõ gì. Đúng cho invariant bạn muốn giữ *kể cả khi* user prompt tự do ngoài mọi skill.
2. **Path-scoped** trong `.claude/rules/*.md` có `paths:` — load trên **Trục 2** khi Claude đọc file khớp. Rẻ, nhưng chỉ fire trong khu vực đó của repo. Đúng cho invariant local theo domain (code billing, auth middleware, một service boundary cụ thể) chỉ quan trọng trong một folder.
3. **Embedded trong body của một skill hoặc agent** — load trên **Trục 2** chỉ khi skill hoặc agent đó được invoke. Rẻ nhất trong ba, nhưng **biến mất ngay lúc user đi off-script**. Đúng khi rule chỉ có ý nghĩa *trong* procedure mà skill chạy — ví dụ *"trong lúc review migration, không bao giờ recommend `DROP TABLE` mà không có backup"*.

Câu hỏi cần hỏi trước khi đặt một rule: *user có thể hợp lý bỏ qua trigger này mà vẫn gặp vấn đề rule đang phòng không?* Nếu có, centralize và trả thuế Trục 1. Nếu không, embed hoặc path-scope.

Điều này quan trọng nhất khi đánh giá các setup nặng framework. Một framework với hàng chục skill được thiết kế tốt có thể embed phần lớn invariant của nó trực tiếp vào body skill / agent và trả gần như bằng 0 thuế mỗi turn — **miễn là user bám theo đường ray**. Ngay khi họ free-form vào một chủ đề framework không lường trước, các invariant embedded không load, và session âm thầm fallback về bất cứ `CLAUDE.md` base hay rule centralized nào đang có. Trên đường ray, setup trông vững như bàn thạch; ngoài đường ray, user thực chất đang làm việc trong một session Claude Code *không rule nào cả* — và họ sẽ không được cảnh báo khi bước qua ranh giới đó.

Hệ quả thực tế: trước khi commit một rule vào body của skill, hãy hỏi *"chuyện gì xảy ra nếu user làm việc trên code này mà không invoke skill?"* Nếu câu trả lời là "rule âm thầm ngừng tồn tại", đó là tín hiệu để hoặc (a) centralize rule thay vì vậy, (b) path-scope để nó fire trên file read thay vì skill invocation, hoặc (c) đỡ nó bằng một hook để enforcement không phụ thuộc model có thấy rule hay không.

### Skill như template: body chung + context theo task

Một sai lầm hấp dẫn khi customize Claude Code là viết một skill cho mỗi kịch bản business — `/billing-migration-planner`, `/user-onboarding-reviewer`, `/invoice-schema-validator` — cho đến khi có skill cho mọi workflow trong công ty. Mỗi description khi đó mang noun đặc thù project trên Trục 1 mãi mãi, và mỗi project mới đẻ ra một skill mới.

Framework nghiêm túc không làm kiểu đó. Họ giữ body skill **generic** — một planner, một executor, một reviewer, một debugger — và đẩy kiến thức đặc thù business vào một **file context bên ngoài mà skill đọc tại runtime**. Skill là executable; file context là data. Chạy chung lại tạo ra hành vi chuyên biệt, mà không nửa nào trong cặp cần biết chi tiết của nửa kia tại thời điểm viết.

Đây là *tách code khỏi data* — một pattern mọi backend engineer đã dùng rồi (Docker image + compose env, React component + props, CLI tool + config). Áp vào Claude Code, nó cho bạn skill giữ được tính committable, share được giữa team, và không phình ra khi số project tăng. Context per-project (`./.planning/PROJECT.md`, `./.task/context.md`, bạn chọn convention nào cũng được) ở lại local trong project và tiến hóa theo nhịp riêng của nó.

Phân bổ chi phí sạch sẽ: description của skill ở nguyên trên **Trục 1** bất kể bạn chạy bao nhiêu project, và file context chỉ chạm **Trục 2** khi skill thực sự được invoke. Thêm một project mới thêm 0 thuế mỗi turn; thêm một *kiểu* workflow mới thì có.

Pattern này có một điểm cần cẩn thận: nó phụ thuộc hoàn toàn vào việc body skill biết *nơi để tìm* file context. Nếu convention là implicit — *"đọc plan trong `.planning/`"* trong một số skill, *"load task từ `./.task/`"* trong skill khác — bạn sẽ nhận silent failure: skill invoke, cố đọc một path không tồn tại, và bịa thay vì dừng. Phiên bản hoạt động được của pattern này đặt tên convention cho path context để cả framework tôn trọng, và tốt hơn nữa là nhắc tới nó trong description của skill để user biết phải chuẩn bị gì trước khi invoke.

Khi đọc thư mục `.claude/` của người khác, hai câu hỏi mở ra mô hình composition của họ: **(1) body skill có đọc state bên ngoài không? (2) nếu có, convention nào báo skill biết nơi tìm?** Một framework trả lời cả hai sạch sẽ có thể tái dùng cho mọi project bạn có. Một framework embed chi tiết project trực tiếp vào body skill là framework cuối cùng bạn sẽ fork.

### Artifact như context bền vững — state sống qua `/compact`

Có một mở rộng tinh tế hơn của pattern template đáng đặt tên riêng. Khi một skill hoặc workflow ghi một file như `PLAN.md`, `CONTEXT.md`, hay `DECISIONS.md` vào một convention path, file đó trở thành **context mới cho command tiếp theo đọc**. GSD làm việc này khắp nơi: `/gsd:plan-phase` tạo ra `PLAN.md`, rồi `/gsd:execute-phase` đọc lại tươi ở lần chạy kế; `/gsd:discuss-phase` tạo ra `CONTEXT.md`, planner đọc trước khi lên kế hoạch. Framework không chỉ *tiêu thụ* authored state (CLAUDE.md, rule, skill body) — nó đang *sinh* ra state của chính mình và tin vào đĩa để mang tiếp.

> **Deep dive:** [`docs/gsd-walkthrough.md`](./docs/gsd-walkthrough.md) đi hết pattern này end-to-end trên một project thật — mỗi command, mỗi lần spawn agent, mỗi artifact được ghi, và cách state chảy qua ranh giới `/compact`.

Đây là câu trả lời cơ học sạch nhất cho vấn đề `/compact` ở Mục 1. `/compact` viết lại message history; nó không viết lại file trên đĩa được. Ba turn research và thảo luận cô đặc thành một `PLAN.md` 500 dòng sống sót qua mọi lần compaction — và mọi session mới, vì `Read` thì rẻ và deterministic. Phân biệt đáng vạch ra: **authored state** (CLAUDE.md, rule, skill) được viết một lần và tiêu thụ mãi mãi; **generated state** (plan, phase context, research summary, decision log) do Claude viết và chính Claude đọc lại sau. Convention path là thứ cho phép chúng đồng tồn tại — framework biết nơi tìm generated state, và generated state không phình Trục 1 vì nó không bao giờ được load trừ khi có command tham chiếu.

**Làm gì khi không có framework.** Pattern này không đòi GSD, cũng không đòi bất kỳ framework nào. Phiên bản tối thiểu khả dụng là một file `./PLAN.md` duy nhất (hoặc `./docs/active/plan.md`, convention nào bạn thích cũng được) commit vào repo, thói quen bảo Claude đọc lúc đầu session, và thói quen update lúc cuối session. Thế là đủ. Framework tự động hóa kỷ luật; kỷ luật work kể cả không có framework. Nếu bạn đang vibe-code session dài và cảm thấy các triệu chứng ở Mục 1, thêm một file này là nâng cấp durable-state rẻ nhất có — nó work vì cùng lý do `.planning/` của GSD work: đĩa sống lâu hơn `/compact`.

**Vì sao planning có đòn bẩy ngoại cỡ.** Nhìn flow top-level của GSD — *plan → execute → verify* — và để ý planning được giao nhiều agent nhất, nhiều gate nhất, nhiều iteration nhất (discuss-phase, plan-checker, revision loop). Execution, so sánh, gần như cơ học: đọc plan, áp dụng. Sự bất đối xứng đó có chủ đích. Planning là chỗ *suy nghĩ đắt đỏ* diễn ra — research, phân tích tradeoff, sắp thứ tự phụ thuộc — và *đầu ra* là một artifact cô đặc, đọc lại được. Một khi artifact đó tồn tại, execution đủ deterministic để chạy trên model yếu hơn, rẻ hơn mà không drift, vì việc suy nghĩ khó đã xảy ra trên đĩa rồi.

Vibe coding đảo ngược điều này. Không có artifact plan, mỗi turn execution đồng thời kiêm re-planning — quyết định ở turn 10 bị âm thầm diễn giải lại ở turn 40, research làm một lần được làm lại. Execution intelligence phải gánh toàn bộ tải, việc đó work trong lúc model còn mạnh và context còn tươi, và bắt đầu gãy ngay khi một trong hai điều kiện đó fail. Khái quát hóa: **càng nhiều suy nghĩ của bạn sống trong durable artifact, bạn càng ít phụ thuộc vào việc model giữ nó trong attention.** Đó mới là lý do thật sự khiến các fact ở Mục 3 về attention và window cắn nặng nhất trong chế độ vibe-code — không có chỗ nào khác cho suy nghĩ trú ngụ.

Điều này đặt lại chi phí của việc adopt một framework. Một vibe-coder chưa có thói quen planning mà với tay tới GSD trả *hai lớp chi phí chồng lên nhau*: **(1) chuyển dịch thói quen** — học cách plan trước khi execute, externalize quyết định, nghĩ theo phase — và **(2) bề mặt của framework** — command, convention, tên artifact, role của agent. Chỉ cái thứ hai mới là độ phức tạp của chính framework; cái đầu là thay đổi workflow mà framework không dạy được, chỉ có thể *thưởng* cho. Ai đã giữ một `./PLAN.md` viết tay chỉ trả chi phí (2), và phần đó nông — họ chỉ đang học nơi framework kỳ vọng kỷ luật sẵn có của họ sống. Đây là một cách đọc hợp lý tại sao người ta thử framework, bật ra, rồi kết luận *"framework không hợp với tôi"* — họ đang trả cả hai chi phí cùng lúc và gán nhầm friction cho công cụ. Một trình tự hợp lý là xây thói quen planning trước trên `./PLAN.md` viết tay, rồi tốt nghiệp sang framework khi nó đã thành phản xạ — cách đó bạn trả chi phí (2) trên nền thói quen đã có, không phải trên nền workflow đang sắp xếp lại bên dưới bạn. Con đường phòng thủ được khác là để chính framework là người huấn luyện: các gate của nó (`plan-phase` trước `execute-phase`, plan-checker, decision tracking) rèn hình dạng đó mỗi session, cách này work nếu bạn chịu được trả cả hai chi phí cùng lúc và kiềm chế không free-form quanh các gate khi chúng thấy bất tiện. Không đường nào là phổ quát; điểm cốt là biết bạn đang trả chi phí nào, và không nhầm thay đổi workflow với độ phức tạp framework.

### Bốn quyết định, không phải một

Trước khi chạy cái cây bên dưới, để ý rằng thứ trông như một câu hỏi placement thực ra là bốn, và chúng tương tác với nhau. Với bất kỳ mẩu kiến thức nào bạn định persist, bạn thực ra đang quyết định:

1. **Primitive nào** — CLAUDE.md, rule, skill, hook, memory, subagent.
2. **Nếu là rule, text của nó đặt ở đâu** — centralized (Trục 1), path-scoped (Trục 2 trên file read), hay embedded trong body skill (Trục 2 trên invocation).
3. **Nếu là skill, body generic + context per-task, hay body đặc thù business** — cái đầu scale qua nhiều project; cái sau không.
4. **Trigger** — explicit `/command`, implicit auto-trigger, hay scope bằng `paths:`.

Mỗi lựa chọn đổi tradeoff của ba cái còn lại. Một rule embedded trong body skill với implicit trigger chỉ fire khi model quyết định invoke — coverage gắn liền với description engineering. Đổi trigger sang explicit và coverage gắn với thói quen user. Centralize rule và coverage được đảm bảo, nhưng bạn trả thuế Trục 1 kể cả khi rule không liên quan. Không có bữa trưa miễn phí — chỉ có compromise nào hợp với failure mode của bạn.

### Decision tree — "kiến thức này sống ở đâu?"

Trigger trả lời *khi nào* cái này fire. Placement trả lời câu trước đó — *primitive nào* một mẩu kiến thức thuộc về ngay từ đầu. Cái đó xứng đáng có một cây. Bắt đầu từ trên; match đầu tiên thắng.

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

Nếu không match gì, nhiều khả năng bạn không cần persist nó. Cây là kỷ luật tối thiểu — mọi thứ trong repo này đều là nỗ lực biến nó thành phản xạ.

---

## 6. Đọc bất kỳ framework nào, tự mình

Tất cả những gì trong hướng dẫn này đều đang luyện một kỹ năng: khả năng nhìn vào một setup Claude Code — của bạn hay của ai khác — và thấy thực sự cái gì đang diễn ra bên dưới.

Chọn một framework phổ biến bất kỳ. ccpm. get-shit-done. oh-my-claudecode. Một bundle *"N skill × M agent"* bạn tìm thấy trên GitHub tuần trước. Không cái nào có thể bỏ qua các primitive mà hướng dẫn này đã trình bày. Nếu một framework ship cái gì đó thực sự ảnh hưởng hành vi Claude, cái đó phải — về mặt cơ học, không có đường nào khác — là một `CLAUDE.md`, một `.claude/rules/*.md`, một `.claude/skills/*/SKILL.md`, một `.claude/hooks/*.sh`, hoặc một memory / subagent. Không có primitive thứ bảy bí mật. Mọi framework rốt cuộc là một đống markdown và shell script bơm context cho Claude, từng turn một, qua cùng một nhúm cơ chế bạn giờ đã hiểu.

Một khi bạn thấy điều này, bạn có thể đọc thư mục `.claude/` của bất kỳ framework nào như đọc source code. Nhìn description của các skill — chúng có đang cạnh tranh attention Trục 1 một cách cẩn trọng, hay phình ra startup prefix? Rule có path-scope bằng `paths:`, hay luôn-load? `disable-model-invocation` có được dùng ở chỗ có side effect không? Hook có thực sự enforce cái gì, hay chỉ trang trí? Bạn không cần README của framework nói liệu nó có hợp *codebase của bạn* không — bạn có thể tự chấm điểm, primitive by primitive.

Bạn làm gì sau đó là việc của bạn. Một số người đọc sẽ thấy một framework hợp với codebase đủ tốt để adopt với vài tweak nhỏ. Người khác sẽ coi framework như *reference implementation* — copy những hình dạng phù hợp, bỏ phần còn lại, tự viết những gì thiếu. Người khác nữa sẽ quyết định rằng những quirk đặc thù của repo xứng đáng có skill và rule viết từ đầu. Cả ba đường đều bảo vệ được. Cái chung của chúng là người ra quyết định giờ có thể thấy cái gì thực sự đang ở trước mặt mình — và đó là kỹ năng mà hướng dẫn này cố gắng xây.

Một hệ luận về *khi nào* một framework đáng với tay tới. Framework là kiến trúc context đã build sẵn — nó sinh lời khi hình dạng nó encode sẽ được tái dùng, và đánh thuế bạn khi không. Hai kiểu workload cần phân biệt:

- **Công việc thăm dò / một lần.** Bài bạn chưa giải trước đây, task không có pattern tái dùng, cuộc điều tra mà hình dạng đang được khám phá dần. Context nhỏ và đặc thù cho task. Overhead framework gần như luôn thua ở đây — bạn trả thuế attention mỗi turn (Mục 2) và friction ánh xạ pipeline cho đường ray sẽ không bao giờ chạy lại. Một prompt free-form sắc bén với đúng file trong scope thắng framework dễ dàng.
- **Công việc lặp lại, đi theo hình dạng.** Viết unit test theo format nhà mình, scaffold service mà cả năm cái trông giống nhau, migrate mọi file trong một directory qua cùng một transform, chạy review pass kiểm tra cùng mười hai thứ mỗi lần. Hình dạng đã biết và lặp lại; giải thích lại mỗi turn là chi phí thật. Ở đây hình dạng cố định của framework *chính* là đòn bẩy — một `/command` encode các bước cộng với một rule/skill encode format trả lại nhanh.

Dấu hiệu nhận biết đơn giản: bạn có viết runbook cho loại công việc này không? Nếu có, framework — hoặc tối thiểu một skill + rule tùy biến — đáng đồng tiền. Nếu không, bạn đang mua đường ray cho một task không cần, và nước đi trung thực là một prompt tốt, không phải một pipeline.

Cùng cái gate đó fix một phản xạ xấu hơn trong session: **nhồi nhét hoảng loạn (panic-cramming)**. Khi Claude gặp edge case và bạn fix, bản năng là nhét bài học vào `CLAUDE.md` phòng khi nó tái diễn — triệu chứng thứ ba của Mục 1, nghĩa địa. Câu hỏi trung thực y hệt: *cái này có tái diễn không?* Nếu không, fix sống trong diff; model không cần mang câu chuyện đi tiếp, và codify một one-off là trả thuế mỗi turn cho một sự kiện sẽ không xảy ra nữa. Nếu có, edge case xứng có một nơi bền vững — *ở đâu chính xác* là quyết định của Mục 5, không phải của đoạn này — nhưng gate để *hỏi ở đầu* là "đây là một hình dạng, hay một sự cố đơn lẻ?" Phần lớn bullet trong nghĩa địa tồn tại vì ai đó bỏ qua gate này và codify one-off thành rule.

Một hệ luận thực tế nếu bạn adopt framework: **invoke `/commands` và skill của nó *explicit*, ở đúng bước mỗi cái được thiết kế cho**. Framework không hook được mọi thứ — hook chỉ fire trên file event và tool call, không trên *ý định*, nên bất cứ thứ gì tác giả không thể nối vào trigger deterministic sẽ sống sau một entry point explicit. Auto-triggering implicit là đường dễ fail từ Mục 5: description tranh attention Trục 1 mỗi turn, và `/compact` âm thầm bỏ. Cách đáng tin để có hành vi mà framework được thiết kế để tạo ra là gọi đúng command ở đúng bước, theo cách tác giả đã đi dây pipeline — loop `discuss → plan → execute → verify → ship` của `get-shit-done` là ví dụ sạch: mỗi phase là một slash command riêng, và skip một phase bằng free-form chat nghĩa là guard rail của phase đó không load. Free-form đi qua cùng task và bạn trượt ra khỏi đường ray mà không nhận ra — các invariant của framework không load, và session fallback về bất cứ gì trong `CLAUDE.md` base. Framework không phải background service; nó là một bộ entry point bạn phải thực sự dùng.

Cú chuyển này đi xa hơn chuyện đọc framework. Một khi bạn biết mọi prompt cạnh tranh trong một budget attention dùng chung, mỗi dòng `CLAUDE.md` là thuế mỗi turn, và `/compact` là diễn giải lại chứ không phải tha thứ, sẽ khó hơn để giao mọi thứ cho model và hy vọng — với Claude Code hôm nay, với tool agentic nào thay nó ngày mai. Prompt nông và ý định mơ hồ luôn là ván cược; bạn giờ biết tỷ lệ. Bạn không phải ngừng delegate. Bạn phải ngừng giả vờ delegation diễn ra trong chân không. **Bạn là kiến trúc sư của cái model thấy; model là bộ não làm việc với thứ bạn đưa cho nó.** Sự cộng tác đổ vỡ đúng lúc hai vai đó bị nhầm lẫn.

Từ đây, bạn tự đọc.

---

## 7. Sắp tới — và repo này KHÔNG phải gì

### Sắp có

- [ ] `docs/anti-patterns.md` — 6 kiểu đặt sai chỗ phổ biến nhất
- [ ] `docs/lifecycle.md` — khi nào promote memory → rule, demote CLAUDE.md → rules/
- [ ] `examples/` — một ví dụ tối giản, đã sanitize cho mỗi primitive
- [ ] Case study — một flow thật (viết test) chia ra 5 layer, kèm reasoning

Welcome issue và PR. Nếu bạn không đồng ý với một placement, mở issue — điểm chính là làm nổi bật các tradeoff.

### Ngoài phạm vi

- Không phải list "awesome-claude". Thứ đó có rồi.
- Không phải tutorial cài Claude Code. Xem [docs chính thức](https://docs.claude.com/en/docs/claude-code).
- Không phải tip prompt-engineering. Ngoài scope.
- Không ý kiến về lựa chọn model. Quá biến động.

### Kêu gọi đóng góp — một dashboard context sống

Thesis trung tâm ở đây là **hiểu Claude đang biết gì ngay lúc này là một kỹ năng, và công cụ để nhìn thấy điều đó đang bị đầu tư thiếu.** `/context` là view tốt nhất hiện có, và nó là một đống text. Có một khoảng trống có thể xây giữa nó và một thứ thực sự hữu ích — session JSONL đã chứa ~80% dữ liệu cần để parse, gán chi phí, và visualize mọi lần context load theo thời gian thực. RFC đầy đủ, build plan từng bước (MVP → TUI → dashboard → team analytics), câu hỏi design mở, và cách tham gia: [`docs/dashboard-rfc.md`](./docs/dashboard-rfc.md).

---

## License

MIT — xem [LICENSE](./LICENSE).
