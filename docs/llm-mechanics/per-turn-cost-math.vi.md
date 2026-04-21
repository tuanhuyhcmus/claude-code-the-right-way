# Chi phí thực mỗi turn của một session Claude Code

*[English](./per-turn-cost-math.md) · **Tiếng Việt***

*Vì sao thêm một skill không bao giờ miễn phí — và cách tính một session dài thực sự tốn bao nhiêu.*

Tài liệu này làm phần toán học đằng sau khẳng định trong README chính: **mọi primitive bạn load đều có chi phí định kỳ mỗi turn, không phải chi phí cài đặt một lần.** Nếu bạn từng thắc mắc vì sao một session dài với nhiều skill "có cảm giác đắt", đây là lời giải thích định lượng.

Con số ở đây dùng giá **Claude Opus 4.7** do Anthropic công bố tại thời điểm viết. Giá có thể thay đổi; hãy đối chiếu với [trang pricing chính thức](https://platform.claude.com/docs/en/about-claude/pricing) trước khi coi chúng là ràng buộc.

---

## Bước 1 — Bốn mức giá bạn cần biết

Anthropic tính các mức giá khác nhau cho các loại input token khác nhau. Với Opus 4.7:

| Loại token                 | Giá mỗi 1M token    | Hệ số so với base   |
| :------------------------- | :------------------ | :------------------ |
| **Regular input**          | $5.00               | 1.0×                |
| **Cache write (5-minute)** | $6.25               | 1.25×               |
| **Cache write (1-hour)**   | $10.00              | 2.0×                |
| **Cache read**             | $0.50               | 0.1×                |
| **Output**                 | $25.00              | 5.0× (để tham chiếu)|

Hai hệ quả quan trọng:

- Một **cache hit rẻ hơn một bậc độ lớn** so với cache miss (0.1× vs. 1.0×).
- Một **cache write đắt thêm 25%** so với giá input thường. Bạn trả một khoản premium một lần để write, rồi read rẻ trong suốt vòng đời của cache.

Kinh tế học của Claude Code xoay quanh việc **giữ nội dung đắt tiền được cached**, không phải giảm kích thước nội dung. **Size quan trọng, nhưng *stability* quan trọng hơn.**

---

## Bước 2 — Cái gì được cache, và cái gì invalidate nó

Mọi request Claude Code gửi đến Anthropic đều có cấu trúc đại khái là:

```
<tools>           ← ít khi đổi
<system prompt>   ← ít khi đổi
<CLAUDE.md>       ← ít khi đổi
<rules>           ← ít khi đổi (trừ khi path-scoped fire)
<skill descriptions>  ← ít khi đổi
<MEMORY.md>       ← ít khi đổi
<message history> ← lớn lên mỗi turn
<your new prompt>
```

Prompt cache của Anthropic hoạt động theo **prefix matching**: cache nhớ prefix khớp chính xác dài nhất tính từ đầu request của bạn. Mọi thứ đến điểm khác biệt đầu tiên là cache hit.

Điều này nghĩa là:

- **Prefix ổn định, suffix lớn lên:** Nếu tools, system prompt, CLAUDE.md, rules, skills, memory, và message history trước đó đều giống hệt request trước, tất cả chúng được phục vụ như cache hit. Chỉ prompt mới (và phần suffix lớn dần của các turn) tốn full-rate input.
- **Prefix thay đổi = cache miss cho mọi thứ phía sau.** Nếu bạn edit `CLAUDE.md` giữa session, cache prefix bị cắt ở ranh giới `<CLAUDE.md>`. Mọi byte sau điểm đó (rules, skills, memory, toàn bộ message history) phải được viết lại vào cache ở turn tiếp theo. **Một lần edit gần đầu prefix rất đắt; một lần edit gần cuối thì rẻ.**
- **Idle quá lâu = cache hết hạn.** TTL cache mặc định là 5 phút. Nếu không có request nào tái sử dụng cache trong 5 phút, entry bị evict và request tiếp theo viết lại từ đầu. Tier 1-hour tốn 2× để write nhưng cho phép bạn idle lâu hơn mà không phải trả lại.

### Cái gì invalidate cache

| Sự kiện                                          | Invalidate từ…                          | Chi phí ròng                      |
| :----------------------------------------------- | :-------------------------------------- | :-------------------------------- |
| Turn bình thường (cùng prefix, prompt mới)       | Không — full hit                        | Rẻ                                |
| Idle 5+ phút (TTL mặc định)                      | Toàn bộ prefix                          | Viết lại toàn bộ ở turn sau       |
| Edit `CLAUDE.md` hay một rule luôn-load          | Từ `<CLAUDE.md>` trở đi                 | Viết lại prefix + history         |
| Thêm / xóa / edit một skill description          | Từ `<skill descriptions>` trở đi        | Viết lại skills + memory + history|
| Invoke một skill mới giữa session                | Từ điểm inject của skill trở đi         | Viết lại suffix                   |
| Edit `MEMORY.md` (auto-memory append)            | Từ `<MEMORY.md>` trở đi                 | Viết lại memory + history         |
| Path-scoped rule fire (match `paths:` mới)       | Từ điểm inject của rule trở đi          | Viết lại suffix                   |

Quy tắc thực tế: **thay đổi càng nằm sớm trong prefix, turn tiếp theo càng phải viết lại nhiều token.**

---

## Bước 3 — Chi phí của một session dài, ví dụ tính toán

Giả sử một setup thực tế:

- Prefix khởi động (tools + system + CLAUDE.md + rules + memory + skill descriptions): **30,000 tokens**
- Nội dung mới mỗi turn (prompt của bạn + response của Claude + tool result): **~2,000 tokens trung bình**
- Độ dài session: **100 turns**, cách nhau đủ để cache vẫn warm (< 5 phút giữa các turn)

### Scenario A — lý tưởng, cache-warm suốt session

- Turn 1: prefix **cache write** (30k tokens × $6.25/MTok) = **$0.188**
- Turn 2–100: prefix **cache read** (30k × $0.50/MTok) × 99 = **$1.485**
- Input mới mỗi turn (2k × $5/MTok) × 100 = **$1.000**
- **Tổng chi phí input-side: ~$2.67**

Phần lớn chi phí là 2k-token suffix mỗi turn, *không phải* 30k-token prefix. Đó là sức mạnh của prompt caching: 30k ở lại rẻ vì nó được read, chứ không bị re-write.

### Scenario B — edit `CLAUDE.md` ở turn 50

Giữa turn 49 và turn 50, bạn thêm một dòng vào `CLAUDE.md`. Prefix tới cache breakpoint đầu tiên giờ đã khác.

- Turn 1–49: như scenario A.
- Turn 50: prefix **cache miss**, phải viết lại. Đó là 30k + 49×2k history tích lũy ≈ **128k tokens** ở cache-write rate (128k × $6.25/MTok) = **$0.800**. Một lần edit vừa khiến bạn tốn thêm khoảng 80 cents phí cache-write.
- Turn 51–100: quay lại hành vi cache-warm, đọc prefix *mới*.

**Khoản phạt ròng cho một lần edit `CLAUDE.md` giữa session: ~$0.80.**

Nhân với một team 10 kỹ sư, mỗi người edit `CLAUDE.md` một cách tùy hứng vài lần mỗi ngày trong các session khám phá, và con số này thành tiền thật — nhưng quan trọng hơn, đó là *tiền vô hình*. Không ai thấy nó. Hóa đơn chỉ ghi "API usage".

### Scenario C — nghỉ uống cà phê 10 phút giữa session

Giữa turn 50 và turn 51, bạn idle 10 phút. Cache mặc định 5 phút bị evict.

- Turn 51: prefix **cache miss** trên toàn bộ prefix hiện tại + 50 turn history tích lũy — khoảng **130k tokens** phải viết lại. Chi phí: ~$0.81.
- Bạn vừa trả số tiền tương đương một lần edit `CLAUDE.md`, chỉ để uống cà phê.

Đây là lý do **tier cache 1 giờ tồn tại**. Với 2× giá write base, nó đắt hơn mỗi turn để viết, nhưng tự trả cho chính nó sau chỉ một lần cache miss mà nếu không thì bạn đã phải chịu.

### Break-even cho cache 1 giờ

Gọi P là kích thước prefix tính theo token, và T là số cache miss dự kiến bạn sẽ phải gánh dưới policy 5 phút trong một session.

- Policy mặc định: T × P × $6.25/MTok
- Policy 1 giờ: 1 × P × $10.00/MTok + (số lần refresh dự kiến trong một giờ) × P × $0.50/MTok

Tier 1 giờ bắt đầu có lời quanh **T ≥ 2** — tức là, nếu bạn dự kiến sẽ idle quá 5 phút hai lần trong một session, bạn có lời. Với các session interactive dài có các khoảng nghỉ ở tốc độ con người, điều đó gần như luôn đúng.

---

## Bước 4 — Chi phí của từng primitive, phân bổ cho một session

Với phép toán trên, ta có thể định giá cho từng lựa chọn kiến trúc.

Giả sử một session 100-turn cache-warm. Thêm nội dung X vào prefix luôn-load tốn, mỗi session:

- Cache-write cost ở turn đầu (nếu X được thêm mới): `|X| × $6.25/MTok`
- Cache-read cost cho 99 turn còn lại: `99 × |X| × $0.50/MTok`
- Chi phí biên mỗi turn: **`|X| × $0.55/MTok`** (xấp xỉ, cho một session dài cache-warm)

Áp dụng cho các kích thước primitive thực tế:

| Primitive                                                   | Kích thước điển hình | Chi phí mỗi session 100-turn |
| :---------------------------------------------------------- | :------------------ | :------------------------ |
| Một skill description                                       | ~100 tokens         | ~$0.006 (nửa cent)        |
| "N skills trong một framework" × 50 skills                  | ~5,000 tokens       | ~$0.28                    |
| Một `CLAUDE.md` phì đại 400 dòng                            | ~5,000 tokens       | ~$0.28                    |
| Một `.claude/rules/integration-test-common.md` 3,200 tokens | ~3,200 tokens       | ~$0.18                    |
| Một MCP server với 50 tool schemas                          | ~10,000 tokens      | ~$0.55                    |

Tính mỗi session, đây là con số nhỏ. Tính mỗi team mỗi ngày, thì không. Một team 20 người với 10 session mỗi ngày mỗi người: **$0.55 × 50 tools × 20 người × 10 sessions = $55 mỗi ngày** chỉ để mang một MCP server phì đại qua mọi session. Đó là $20k/năm trước khi có ai invoke một tool.

Ngay cả một `CLAUDE.md` phì đại duy nhất nằm trong prefix của mọi session cũng là một khoản thuế định kỳ gần như luôn vô hình với chính những người đang trả nó.

---

## Bước 5 — Điều này nghĩa gì cho thiết kế repo

Phép toán này đóng khung lại một số lựa chọn trông có vẻ miễn phí tại thời điểm viết:

- **Thêm một skill không miễn phí.** Mỗi description bạn giữ trong tier luôn-load tốn token ở mọi turn của mọi session, mãi mãi, dù skill đó có được invoke hay không. `disable-model-invocation: true` thực sự loại bỏ khoản thuế đó — description không bao giờ được thêm vào startup prompt.
- **Phình `CLAUDE.md` đắt theo hai cách.** Thứ nhất, chính kích thước tốn tiền mỗi turn (xem bảng). Thứ hai, edit một `CLAUDE.md` to bust cache cho toàn bộ prefix + message history phía sau, gây ra khoản phạt cache-write tỉ lệ với thời gian session đã chạy.
- **Path-scoped rule là đúng đắn về mặt tài chính** ở nơi chúng áp dụng. Một rule chỉ load khi Claude đụng `src/billing/**` tốn 0 token ở những turn không bao giờ đụng directory đó. Tradeoff: path-scoped rule mất sau `/compact` và reload ở lần match tiếp theo, rẻ nếu match hiếm.
- **Subagent thực sự tiết kiệm tiền cho các tác vụ đọc nặng.** Context window của subagent được tính riêng so với của bạn. Nếu bạn delegate một task research 50k-token cho subagent, cuộc hội thoại cha chỉ trả cho một summary ~1k-token khi trả về, không phải 50k của file read.
- **Tier cache 1 giờ gần như luôn đáng** cho các session interactive. Nếu bạn đang trong một session có bất kỳ khoảng nghỉ ở tốc độ con người nào, bạn sẽ có lời.

Không có điều gì ở đây là lý do để ngừng dùng primitive. Đó là lý do để **budget chúng** — coi việc thêm một skill hay rule như thêm một dependency vào hot loop. Chỉ những cái xứng đáng với chi phí mỗi turn mới nên ở lại trong tier luôn-load. Phần còn lại thuộc về `disable-model-invocation`, scope `paths:`, hoặc `/commands` thủ công.

---

## Nguồn

- [Anthropic Prompt Caching — Pricing and TTL](https://platform.claude.com/docs/en/docs/build-with-claude/prompt-caching)
- [Claude Code — Context window simulation](https://code.claude.com/docs/en/context-window)
- [Claude Code — Skills](https://code.claude.com/docs/en/skills)
- [Anthropic — Pricing](https://platform.claude.com/docs/en/about-claude/pricing)
