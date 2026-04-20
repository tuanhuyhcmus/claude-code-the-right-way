# Self-attention, nói đơn giản

> **Ngôn ngữ:** [English](./self-attention.md) · **Tiếng Việt**

*Vì sao context window 1M token là trần, không phải siêu năng lực.*

Doc này dành cho người dùng Claude Code nhưng chưa đọc transformer paper. Không math. Analogy ở chỗ có thể. Nếu bạn cần rigor, các paper link ở dưới có.

---

## Token là gì?

Token là đơn vị model đọc. Tương đương một từ hoặc một chunk của từ.

- `"Hello world!"` → khoảng 3 token
- Một function definition điển hình → 50–200 token
- Một file Go 2000 dòng → 30–50k token

Mọi thứ model thấy — message của bạn, reply của chính nó, CLAUDE.md, tool definition, file read — đều được convert thành sequence token và concatenate thành một sequence lớn trước khi feed vào.

---

## "Attention" là gì, về trực giác?

Đọc câu này:

> *"The cat sat on the mat because **it** was tired."* (Con mèo ngồi trên tấm thảm vì **nó** mệt.)

Khi não bạn chạm vào **"it"**, nó tự động nhảy ngược lại và link tới **"cat"**, không phải "mat". Việc reach-back-and-link đó là những gì attention làm cho model.

Khi một LLM generate token tiếp theo, nó nhìn qua mọi token đã có trong window và quyết định: *mỗi cái quan trọng bao nhiêu với những gì tôi đang viết ngay bây giờ?* Mỗi token nhận một weight. Tổ hợp có trọng số shape output tiếp theo.

Điều này xảy ra không phải một lần, mà cho mọi token model produce. Và không phải trên một "relevance channel" duy nhất, mà trên nhiều cái song song — các attention head khác nhau chuyên biệt cho các relationship khác nhau (syntax, reference, style, v.v.).

Cơ chế này không phải Claude-specific. Nó là core của mọi LLM hiện đại: GPT, Gemini, Llama, Mistral, DeepSeek, Claude — tất cả đều là transformer, tất cả đều bị ràng buộc bởi nó. Các rule thực tế ở cuối trang này áp dụng bất kể bạn đang dùng model nào.

---

## Nhưng làm sao nó thật sự pick "cat" mà không phải "mat"?

Câu hỏi hay. Cả hai đều là noun. Cả hai đều là grammatical candidate. Model không roll xúc xắc. Hai thành phần làm việc cùng nhau: **những gì model học được trong training**, và **cách attention apply knowledge đó ở inference time**.

### Training: priors đến từ đâu

Trong quá trình training, model đã thấy hàng trăm tỷ câu. Trong các câu đó:

- Pronoun như "it" gần như luôn refer back tới một noun xuất hiện gần đây — nên model học bias về phía token noun-like gần đây.
- Một số noun co-occur nhiều hơn với một số predicate. `"cat" + "tired"` xuất hiện nhiều hơn `"mat" + "tired"` trong văn bản tự nhiên, vì thảm không mệt còn mèo thì có.
- Subject thường là antecedent của pronoun hơn object.

Model không memorize rule explicit ("nếu pronoun, link tới subject"). Nó internalize chúng thành hàng tỷ numerical weight trong neural network — weight làm certain linkage statistically likely hơn các linkage khác trên bất kỳ input mới nào.

Không có training, capability này đơn giản không tồn tại. Một model mới initialize sẽ link "it" tới "cat" và "mat" với xác suất bằng nhau. **Training là nơi knowledge sống.** Attention là cơ chế retrieve mảnh đúng của nó cho một input cụ thể.

### Attention ở inference: Q, K, V

Khi model xử lý input, mỗi token produce ba vector (list số ngắn) encode các facet khác nhau về nó là gì và nó cần gì:

- **Query (Q)** — "tôi đang tìm gì trong prior context?"
- **Key (K)** — "tôi quảng cáo gì về bản thân, để người khác tìm thấy tôi?"
- **Value (V)** — "nếu tôi được select là relevant, tôi đóng góp gì?"

Ba vector này không được viết bằng tay. Chúng được produce bởi các small learned transformation (chính chúng cũng được shape bởi training). Mỗi token có Q, K, và V riêng.

Khi model đến **"it"**, nó produce một Query về cơ bản nói: *"Tôi là pronoun. Tôi đang tìm recent, singular, animate-ish noun có thể là subject của 'was tired'."* Mọi token trước đã publish một Key:

- `cat` — *"singular animate noun, subject của clause hiện tại"*
- `mat` — *"singular inanimate noun, object của preposition"*
- `sat` — *"past-tense verb, predicate"*
- ...

Model compute similarity giữa Query của "it" và Key của mọi token trước. Key của `cat` match gần nhất, nên `cat` nhận attention weight cao nhất. Các Value vector sau đó được combine, weighted bởi các similarity đó, và kết quả là những gì "it" carry forward vào layer computation tiếp theo.

Analogy hoạt động: nghĩ về nó như **fuzzy library catalog search**. Mỗi token trước đã đặt một card trong catalog (Key của nó). Token hiện tại đến với một search question (Query của nó). Model làm fuzzy similarity match và rank tất cả card. Nội dung card thắng (Value của nó) được pull ra và mix vào answer.

### Vậy: training hay attention?

**Cả hai, ở thời điểm khác nhau.**

- *Training* quyết định pattern nào model biết. Một model chưa từng train trên tiếng Anh không thể resolve pronoun tiếng Anh dù attention của nó thông minh thế nào.
- *Attention* quyết định pattern nào trong số đó fire trên input cụ thể hôm nay. Nó là cơ chế retrieval pick cái gì quan trọng ở đây, bây giờ.

Bạn thỉnh thoảng sẽ đọc "attention là cách LLM nghĩ". Đó là compress hai thứ riêng biệt thành một. Chính xác hơn: attention là **cách model apply, ở inference, kiến thức mà training baked in.**

### Vì sao điều này quan trọng cho prompt của bạn

- **Relevance được compute, không phải declared.** Bạn không thể chỉ viết *"pay attention to X"* và mong model tuân theo. Q/K match vẫn chạy trên tất cả trong context. Instruction của bạn giúp chỉ khi nó reshape attention math thật.
- **Noise cạnh tranh attention weight.** Paste một log 500 dòng bên cạnh câu hỏi, và các token từ log đó carry real weight trong computation. Chúng không lịch sự chờ được ignore.
- **Training set priors.** Model train nặng trên Python có attention pattern mạnh cho Python hơn cho, chẳng hạn, Erlang. Đó là lý do Claude đáng kể tốt hơn ở một số language, framework, và tool hơn cái khác — không phải phép thuật, chỉ là training distribution.

---

## Vì sao gọi là "self"-attention?

Vì mọi token attend tới mọi token khác trong cùng sequence. Đây là pairwise relationship giữa tất cả token.

Cho sequence N token, model compute khoảng N × N attention weight:

| Token trong window | Pairwise weight computed  |
| ------------------ | ------------------------: |
| 1,000              | 1,000,000                 |
| 10,000             | 100,000,000               |
| 100,000            | 10,000,000,000            |
| 1,000,000          | 1,000,000,000,000         |

Inference hiện đại dùng các trick engineering để giữ tractable, nhưng shape cơ bản là O(N²): gấp đôi context là gấp bốn lượng việc.

Đó là một lý do context 1M token không "free" ngay cả ở phía vendor — nó là thành tựu kỹ thuật thật để cho chạy được.

---

## Điểm gài: attention là zero-sum pie

Đây là phần thay đổi cách bạn nghĩ về prompt.

Các attention weight một token distribute qua mọi prior token **phải tổng bằng 1**. Đó là softmax — một pie cố định, được chia.

Tưởng tượng một spotlight. Tổng wattage cố định. Nếu bạn chiếu nó vào:

- **10 item** → mỗi cái nhận 10% ánh sáng
- **1,000 item** → mỗi cái nhận 0.1%
- **1,000,000 item** → mỗi cái nhận 0.0001%

Model vẫn "thấy" mọi thứ trong window. Nhưng mỗi token irrelevant thêm vào **giảm weight available cho token relevant**. Trong long context, tín hiệu quan trọng drown in noise.

Đây là lý do cơ chế đằng sau một fact phản trực giác: *một prompt gấp đôi có thể produce answer tệ hơn prompt nửa kích thước, nếu nội dung thêm là noise.* Không phải model "quên" — nó chỉ có ít attention-weight hơn để dành cho cái thật sự matter.

> **Caveat — spotlight là analogy, không phải phương trình.** Model thật có nhiều attention head (hàng chục) và nhiều layer (hàng chục nữa), mỗi cái có softmax riêng. Dilution không tuyến tính như "một spotlight, nhiều item". Một relevant token không thật sự drop xuống 0.0001% weight trong context 1M token — nó vẫn có thể dominate một head hoặc một layer ngay cả khi diluted ở cái khác. Analogy capture *hướng* của effect (nhiều token hơn → ít weight per token trung bình) nhưng overstate *magnitude*. Xem nó như intuition, không phải prediction.

---

## Hiệu ứng "lost in the middle"

Empirically, model làm tốt nhất thông tin gần **start** và **end** của context, và tệ nhất thông tin ở **giữa**. Điều này đã được đo qua nhiều model ([Liu et al. 2023](https://arxiv.org/abs/2307.03172)).

Cụ thể: nếu CLAUDE.md 10000 dòng của bạn có rule critical ở dòng 5000, nó gần vô dụng hơn hữu ích. Model có nó trong context, nhưng attention không thể reliably tìm ra.

> **Caveat — effect có thật nhưng đã yếu đi ở model mới.** Paper 2023 test các model giờ đã mấy generation trước. Model long-context gần đây (bao gồm Claude Opus 4.x) perform tốt hơn đáng kể trên needle-in-a-haystack retrieval so với model Liu et al. đo. Lost-in-the-middle effect chưa biến mất, nhưng magnitude nhỏ hơn graph gốc paper suggest. Interpret nó như *"bias về front-and-end vẫn tồn tại, plan accordingly,"* không phải *"giữa vô dụng."*

Docs Claude Code dù vậy vẫn recommend giữ `CLAUDE.md` dưới ~200 dòng. Threshold đó chủ yếu driven bởi *adherence* (file ngắn produce instruction-following consistent hơn) thay vì raw lost-in-the-middle math, nhưng hai effect point cùng hướng.

---

## Vậy vì sao Claude Opus advertise 1M token?

Vì architecturally model *có thể* attend qua 1M token. Với task hiếm — refactor toàn repo, analyse document pháp lý dài, synthesise qua nhiều file — điều này thật sự valuable.

Nhưng:

- **Can ≠ should.** Dùng full window thường degrade answer quality do attention dilution.
- **1M là trần, không phải workspace.** Working set hiệu quả của bạn nhỏ hơn nhiều.
- **Context lớn hơn là reserve, không phải perf upgrade.** Xem nó như RAM: có 64GB không có nghĩa bạn nên hold mọi file open cùng lúc.

Nếu bạn đang trả thêm cho long-context model và dùng nó cùng cách bạn dùng model 8k-context, bạn đang trả cho capacity bạn không benefit.

---

## Rule thực tế cho Claude Code

Mọi thứ dưới đây follow từ ba cơ chế trên (pairwise attention, zero-sum weight, lost-in-the-middle):

1. **Prompt ngắn hơn thường outperform prompt dài hơn.** Nếu bạn có thể nói trong 50 dòng thay vì 500, hãy làm.
2. **Đặt rule critical gần top** của CLAUDE.md, không chôn giữa.
3. **Dùng path-scoped rule** (`paths:` frontmatter trong `.claude/rules/`) — chỉ load khi Claude đụng file khớp. Ít dilution cho phần còn lại của session.
4. **Dùng skill** cho nội dung chỉ matter khi invoke. Body của chúng không dilute attention cho đến khi Claude thật sự invoke.
5. **Dùng subagent cho heavy read.** Subagent có window fresh riêng — hội thoại parent không bị polluted bởi research của nó.
6. **Coi chừng context rot.** Session dài tích lũy tool output, file read, và compacted summary. Tại một điểm, bắt đầu fresh nhanh hơn tiếp tục.

---

## Muốn đào sâu hơn?

- [**"Lost in the Middle: How Language Models Use Long Contexts"** — Liu et al. 2023](https://arxiv.org/abs/2307.03172) — nghiên cứu empirical đằng sau effect
- [**"Attention Is All You Need"** — Vaswani et al. 2017](https://arxiv.org/abs/1706.03762) — paper transformer gốc (technical)
- [**Claude Code: context window**](https://code.claude.com/docs/en/context-window) — cách Claude Code budget và visualize window trong thực tế
