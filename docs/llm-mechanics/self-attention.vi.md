# Self-attention, giải thích bằng lời thường

*[English](./self-attention.md) · **Tiếng Việt***

*Vì sao context window 1M token là một cái trần, không phải siêu năng lực.*

Tài liệu này dành cho người dùng Claude Code nhưng chưa đọc paper về transformer. Không toán. Dùng analogy ở những chỗ có thể. Nếu bạn muốn rigor, các paper link ở cuối trang có đủ.

---

## Token là gì?

Token là đơn vị mà model đọc. Đại khái là một từ hoặc một mẩu của một từ.

- `"Hello world!"` → khoảng 3 token
- Một function definition điển hình → 50–200 token
- Một file Go 2000 dòng → 30–50k token

Mọi thứ model thấy — tin nhắn của bạn, reply của chính nó, CLAUDE.md, tool definition, file đã đọc — đều được chuyển thành một chuỗi token và nối lại thành một chuỗi lớn duy nhất trước khi feed vào.

---

## "Attention" là gì, theo trực giác?

Đọc câu này:

> *"The cat sat on the mat because **it** was tired."*

Khi não bạn chạm tới **"it,"** nó tự động nhảy ngược và liên kết với **"cat,"** chứ không phải "mat". Cái hành động với-ngược-rồi-nối đó chính là thứ attention làm cho model.

Khi một LLM sinh token tiếp theo, nó nhìn khắp mọi token đã có trong window và quyết định: *mỗi token quan trọng bao nhiêu đối với cái tôi đang viết ngay lúc này?* Mỗi token được gán một weight. Tổ hợp weighted đó định hình output tiếp theo.

Chuyện này xảy ra không phải một lần, mà cho **mỗi** token model sinh ra. Và không chỉ trên một "kênh liên quan" duy nhất, mà trên nhiều kênh song song — các attention head khác nhau chuyên về các quan hệ khác nhau (cú pháp, tham chiếu, phong cách, v.v.).

Cơ chế này không phải đặc thù của Claude. Nó là lõi của mọi LLM hiện đại: GPT, Gemini, Llama, Mistral, DeepSeek, Claude — tất cả đều là transformer, tất cả đều bị ràng buộc bởi nó. Các quy tắc thực dụng ở cuối trang áp dụng bất kể bạn đang dùng model nào.

---

## Nhưng model thực sự *pick* "cat" thay vì "mat" bằng cách nào?

Câu hỏi hợp lý. Cả hai đều là danh từ. Cả hai đều là ứng viên ngữ pháp hợp lệ. Model không tung xúc xắc. Hai thành phần cùng làm việc: **cái model học được trong lúc training**, và **cách attention áp dụng kiến thức đó tại inference time**.

### Training: nơi các prior ra đời

Trong quá trình training, model đã thấy hàng trăm tỉ câu. Trong số các câu đó:

- Các đại từ như "it" gần như luôn tham chiếu ngược về một danh từ vừa xuất hiện gần đó — nên model học được xu hướng thiên về các token kiểu-danh-từ gần nhất.
- Một số danh từ đồng xuất hiện thường xuyên hơn rất nhiều với một số predicate nhất định. `"cat" + "tired"` xuất hiện thường xuyên hơn `"mat" + "tired"` rất nhiều trong văn bản tự nhiên, vì cái thảm không mệt còn con mèo thì có.
- Subject là tiền ngữ của đại từ thường xuyên hơn object.

Model không ghi nhớ các quy tắc một cách tường minh ("nếu đại từ, liên kết với subject"). Nó nội hóa chúng thành hàng tỉ weight số trong neural network — các weight làm cho một số liên kết trở nên *thống kê hơn* so với các liên kết khác trên bất kỳ input mới nào.

Không có training, khả năng này đơn giản là không tồn tại. Một model mới khởi tạo sẽ liên kết "it" với "cat" và "mat" với xác suất bằng nhau. **Kiến thức sống trong training.** Attention là cơ chế rút ra đúng mẩu kiến thức đó cho một input cụ thể.

### Attention tại inference: Q, K, V

Khi model xử lý một input, mỗi token sinh ra ba vector (danh sách số ngắn) encode các khía cạnh khác nhau về việc nó là gì và nó cần gì:

- **Query (Q)** — *"tôi đang tìm gì trong prior context?"*
- **Key (K)** — *"tôi quảng cáo điều gì về bản thân, để người khác tìm thấy tôi?"*
- **Value (V)** — *"nếu tôi được chọn là relevant, tôi đóng góp cái gì?"*

Ba vector này không được viết tay. Chúng được sinh ra bởi các phép biến đổi học được nhỏ (bản thân chúng cũng do training nặn nên). Mỗi token có Q, K, và V riêng của nó.

Khi model tới **"it"**, nó tạo ra một Query đại loại nói: *"Tôi là một đại từ. Tôi đang tìm một danh từ gần đây, số ít, có vẻ là thực thể sống, có thể làm subject của 'was tired'."* Mỗi token trước đó đã publish một Key:

- `cat` — *"danh từ số ít, sống động, subject của mệnh đề hiện tại"*
- `mat` — *"danh từ số ít, vô tri, object của một preposition"*
- `sat` — *"động từ thì quá khứ, predicate"*
- ...

Model tính similarity giữa Query của "it" và Key của từng token trước đó. Key của `cat` khớp gần nhất, nên `cat` được attention weight cao nhất. Các Value vector sau đó được kết hợp, weighted bởi các similarity đó, và kết quả là cái mà "it" mang theo vào layer tiếp theo của computation.

Một analogy khớp: nghĩ về nó như **một lần tìm kiếm fuzzy trong thẻ mục lục thư viện**. Mỗi token trước đó đã đặt một tấm thẻ trong catalog (Key của nó). Token hiện tại tới với một câu hỏi tìm kiếm (Query của nó). Model thực hiện fuzzy similarity match và xếp hạng tất cả thẻ. Nội dung của thẻ thắng (Value của nó) được rút ra và trộn vào câu trả lời.

### Vậy: training hay attention?

**Cả hai, ở các thời điểm khác nhau.**

- *Training* quyết định model biết những pattern gì, nói chung. Một model chưa bao giờ train trên tiếng Anh không thể resolve đại từ tiếng Anh dù attention của nó có khéo đến đâu.
- *Attention* quyết định những pattern nào trong số đó được kích hoạt trên input cụ thể hôm nay. Nó là cơ chế retrieval chọn ra cái gì quan trọng *ở đây, ngay lúc này*.

Thỉnh thoảng bạn sẽ đọc thấy câu "attention is how LLMs think." Đó là cách gộp hai thứ khác nhau làm một. Chính xác hơn: attention là **cách model áp dụng, tại inference, kiến thức mà training đã nướng sẵn vào.**

### Vì sao điều này quan trọng cho prompt của bạn

- **Relevance được *tính*, không phải *khai báo*.** Bạn không thể chỉ viết *"pay attention to X"* và mong model tuân theo. Phép Q/K match vẫn chạy trên mọi thứ trong context. Instruction của bạn chỉ giúp nếu nó thực sự reshape được phép toán attention.
- **Noise cạnh tranh attention weight.** Paste một log 500 dòng bên cạnh câu hỏi của bạn, và các token từ log đó mang weight thật trong computation. Chúng không lịch sự chờ bị bỏ qua.
- **Training đặt prior.** Một model train nặng trên Python có attention pattern mạnh hơn cho Python so với, ví dụ, Erlang. Đó là lý do Claude đáng kể giỏi hơn ở một số ngôn ngữ, framework, và tool so với các cái khác — không phải ma thuật, chỉ là phân phối training.

---

## Vì sao gọi là "self"-attention?

Vì mỗi token attend tới mọi token khác trong *cùng sequence*. Đó là một quan hệ pairwise giữa tất cả các token.

Với một sequence N token, model tính khoảng N × N attention weight:

| Số token trong window | Số weight pairwise phải tính |
| --------------------- | ---------------------------: |
| 1,000                 | 1,000,000                    |
| 10,000                | 100,000,000                  |
| 100,000               | 10,000,000,000               |
| 1,000,000             | 1,000,000,000,000            |

Inference hiện đại dùng nhiều thủ thuật engineering để giữ cái này tractable, nhưng hình dạng cơ bản là O(N²): nhân đôi context xấp xỉ nhân bốn lượng công việc.

Đây là một lý do vì sao context 1M-token không "miễn phí" kể cả ở phía vendor — làm cho nó chạy được chút nào là một kỳ công kỹ thuật thực sự.

---

## Bẫy: attention là một cái bánh tổng bằng không

Đây là phần sẽ thay đổi cách bạn nghĩ về prompt.

Các attention weight mà một token phân phối trên mọi token trước **phải cộng lại thành 1**. Đó là softmax — một cái bánh cố định, bị chia ra.

Hãy tưởng tượng một cái đèn pha. Tổng công suất cố định. Nếu bạn chiếu nó vào:

- **10 item** → mỗi item nhận 10% ánh sáng
- **1,000 item** → mỗi item nhận 0.1%
- **1,000,000 item** → mỗi item nhận 0.0001%

Model vẫn có thể "thấy" mọi thứ trong window. Nhưng mỗi token không liên quan thêm vào **giảm weight khả dụng cho các token relevant**. Trong context dài, signal quan trọng chết đuối trong noise.

Đây là lý do cơ học đằng sau một fact phản trực giác: *một prompt dài gấp đôi có thể cho câu trả lời tệ hơn một prompt ngắn bằng nửa, nếu phần thêm vào là noise.* Không phải model "quên" — nó chỉ có ít attention weight hơn để dành cho thứ thực sự quan trọng.

---

## Hiệu ứng "lost in the middle"

Về mặt thực nghiệm, model làm tốt nhất với thông tin gần **đầu** và **cuối** context, và tệ nhất với thông tin ở **giữa**. Điều này đã được đo trên nhiều model ([Liu et al. 2023](https://arxiv.org/abs/2307.03172)).

Cụ thể: nếu CLAUDE.md 10,000 dòng của bạn có một rule quan trọng ở dòng 5,000, nó gần như vô dụng hơn là hữu ích. Model có nó trong context, nhưng attention không thể tin cậy tìm ra nó.

Đây là lý do các docs Claude Code khuyến nghị giữ CLAUDE.md dưới ~200 dòng. Đó không phải ưu tiên phong cách tùy tiện — đó là work-around cho một điểm yếu đã biết của architecture.

---

## Vậy vì sao Claude Opus quảng cáo 1M token?

Vì về mặt architecture, model *có thể* attend trên 1M token. Với những task hiếm — refactor cả repo, phân tích tài liệu pháp lý dài, tổng hợp qua nhiều file — điều này thực sự có giá trị.

Nhưng:

- **Có-thể ≠ nên.** Dùng full window thường làm giảm chất lượng câu trả lời vì attention bị pha loãng.
- **1M là cái trần, không phải workspace.** Working set hiệu quả của bạn nhỏ hơn rất nhiều.
- **Context to hơn là khoản dự trữ, không phải nâng cấp hiệu năng.** Coi nó như RAM: có 64GB không có nghĩa là bạn nên mở mọi file cùng lúc.

Nếu bạn đang trả thêm tiền cho một model long-context và dùng nó y như cách bạn dùng model context 8k, bạn đang trả tiền cho capacity mà bạn không hưởng lợi.

---

## Quy tắc thực dụng cho Claude Code

Mọi thứ bên dưới chảy ra từ ba cơ chế phía trên (pairwise attention, zero-sum weight, lost-in-the-middle):

1. **Prompt ngắn thường vượt mặt prompt dài.** Nếu nói được trong 50 dòng thay vì 500, hãy làm vậy.
2. **Đặt các rule quan trọng ở gần đầu** CLAUDE.md, không chôn ở giữa.
3. **Dùng path-scoped rule** (`paths:` frontmatter trong `.claude/rules/`) — chỉ load khi Claude đụng file khớp. Ít pha loãng hơn cho phần còn lại của session.
4. **Dùng skill** cho nội dung chỉ quan trọng khi được invoke. Body của chúng không pha loãng attention cho đến khi Claude thực sự invoke.
5. **Dùng subagent cho các lần đọc nặng.** Một subagent có window mới của riêng nó — cuộc hội thoại parent không bị ô nhiễm bởi research của nó.
6. **Để ý context rot.** Session dài tích lũy tool output, file đọc, và các bản tóm tắt sau compaction. Đến một lúc nào đó, bắt đầu lại từ đầu nhanh hơn là tiếp tục.

---

## Muốn đào sâu hơn?

- [**"Lost in the Middle: How Language Models Use Long Contexts"** — Liu et al. 2023](https://arxiv.org/abs/2307.03172) — nghiên cứu thực nghiệm đằng sau hiệu ứng này
- [**"Attention Is All You Need"** — Vaswani et al. 2017](https://arxiv.org/abs/1706.03762) — paper transformer gốc (kỹ thuật)
- [**Claude Code: context window**](https://code.claude.com/docs/en/context-window) — cách Claude Code budget và visualize window trong thực tế
