# Claude Code, the right way

> **Ngôn ngữ:** [English](./README.md) · **Tiếng Việt**

**Mục tiêu của guide này: giúp bạn tổ chức setup Claude Code — `CLAUDE.md`, rule, skill, hook, memory, subagent — để chúng làm việc *cùng* cách model thật sự xử lý request của bạn, chứ không phải chống lại nó. Bạn đọc xong sẽ có một mental model để quyết định mỗi mảnh knowledge của project nên sống ở đâu.**

Đa số người dùng Claude Code đối xử với context một cách thụ động: mở session, làm việc, để file read, tool output, skill, và hội thoại tự tích lũy. Cách này ổn với session ngắn dùng frontier model. Nó gãy khi session dài, khi dùng model yếu, hoặc khi bạn stack nhiều extension mà không ai đo tổng context.

**Mindset mà guide này đề xuất: xem mọi skill, rule, hook, agent, và MCP server trong `.claude/` của bạn như một *công cụ chủ động để quản lý attention của Claude* — không phải bộ sưu tập tiện lợi thụ động.** Mỗi item bạn thêm hoặc giúp model focus, hoặc feed thêm noise. Phần cơ chế phía dưới nói rõ cái nào là cái nào, và phần còn lại của guide biến điều đó thành quyết định placement cụ thể.

Cụ thể, mindset này nhìn như sau:

- Trước khi **thêm** một skill, hỏi: nó tốn bao nhiêu context, và không có nó thì model mất gì?
- Trước khi **always-load** một rule, hỏi: path-scope với `paths:` có rẻ hơn không?
- Khi Claude cứ quên một MUST-NOT qua các turn, **promote** nó thành hook thay vì nhấn mạnh lại bằng prose.
- Khi một task cần đọc hàng nghìn dòng source, **push** việc đọc đó vào subagent để main session giữ sạch.
- Khi session bắt đầu cảm thấy chậm hơn hoặc tệ hơn, việc đầu tiên cần check không phải là model — mà là `/context`, xem cái gì đã lấp đầy window.

Đây là thói quen có leverage cao nhất trong Claude Code. Là thứ phân biệt rõ nhất giữa setup cảm giác sắc bén và setup cảm giác ì ạch, trên cùng model và cùng repo.

Guide này mang quan điểm cá nhân. Nó gần với một bài essay kiểu *"đây là những gì tôi nghĩ hầu hết mọi người đang làm sai"* hơn là tài liệu tham khảo trung lập. Nếu bạn cần trung lập, [Claude Code docs](https://docs.claude.com/en/docs/claude-code) tốt hơn.

> Viết dựa trên sử dụng hàng ngày trên một Go monorepo cỡ trung. Pattern trên TypeScript / Python / Rust / Elixir gần như chắc chắn khác — coi mọi thứ ở đây là điểm khởi đầu, không phải quy luật phổ quát. Sample size: một.

<details>
<summary><strong>Trước tiên, ôn lại nhanh</strong> — click để mở rộng nếu bạn cần nhắc lại <code>CLAUDE.md</code>, rule, skill, hook, memory, và subagent thật ra là gì. Bỏ qua nếu đã biết.</summary>

<br/>

**`CLAUDE.md`** — file markdown ở root repo (hoặc subdirectory). Claude đọc mỗi turn, nên là nơi đúng cho behavioral guideline áp dụng mọi task ("ưu tiên edit hơn tạo file", "không thêm comment trừ khi được yêu cầu"). [Official docs](https://docs.claude.com/en/docs/claude-code/memory).

**Rules** (`.claude/rules/*.md`) — file markdown dưới `.claude/rules/` mà Claude Code auto-discover recursive và load cùng priority với `.claude/CLAUDE.md`. Dùng để tách project invariant ("MUST / MUST NOT") ra khỏi `CLAUDE.md` bloated. YAML frontmatter `paths:` scope rule tới file pattern cụ thể để chỉ load khi Claude đụng file khớp — tốt để giảm context noise. Rule user-level ở `~/.claude/rules/`. [Official docs](https://code.claude.com/docs/en/memory#organize-rules-with-claude/rules/).

**Skills** (`.claude/skills/<name>/SKILL.md`) — workflow có tên, trigger-based, Claude có thể invoke. Mỗi skill có description nói Claude khi nào dùng, cộng với procedural step bên trong. Tốt cho "khi user nói X, làm Y₁ → Y₂ → Y₃". [Official docs](https://docs.claude.com/en/docs/claude-code/skills).

**Hooks** (`.claude/hooks/*.sh` + `settings.json`) — shell script mà harness Claude Code tự chạy ở lifecycle event (pre-tool, post-tool, user-prompt-submit, v.v.). Exit code khác 0 có thể block action. Đây là layer *machine enforcement* — thứ duy nhất ngăn Claude vi phạm rule ngay cả khi rule đã load trong context. [Official docs](https://docs.claude.com/en/docs/claude-code/hooks).

**Memory** (`~/.claude/projects/<project>/memory/*.md`) — user-local, persist qua session. Claude tự ghi vào đây để nhớ fact về bạn, project, preference. *Không* cho những thứ mà code hoặc `git log` trả lời được. [Official docs](https://docs.claude.com/en/docs/claude-code/memory).

**Subagents** (invoke qua `Agent` tool) — instance Claude mới với context window riêng, spawn để xử lý task có scope. Parent chỉ thấy summary cuối. Dùng để isolate heavy read (hàng nghìn dòng) hoặc parallel independent work. [Official docs](https://docs.claude.com/en/docs/claude-code/sub-agents).

</details>

---

## Cơ chế dưới mui xe

Để áp dụng mindset tốt, bạn cần ba fact cơ chế (cộng hai nuance) về cách LLM và Claude Code thật sự xử lý một request. Mọi quyết định placement trong guide này là hệ quả của chúng. Nhảy tới [các primitive](#các-primitive-tóm-lược) nếu bạn đã biết cơ chế; đọc tiếp nếu muốn hiểu lý do đằng sau.

### Fact 1 — Context window có giới hạn, và được chia sẻ

Model chỉ "thấy" được một số token cố định tại một thời điểm — gọi là context window. Claude Opus 4.7 tối đa 1M token. Nghe to khủng khiếp. Nhưng không phải, vì budget này là **shared**, không phải dedicated cho câu hỏi của bạn.

Tất cả dưới đây cùng cạnh tranh 1M token trên mỗi turn:

- System prompt của Claude Code (vài nghìn token chỉ để define behavior)
- Mọi JSON schema của tool — built-in tool, MCP tool, agent definition (thường 10–20k token cho setup điển hình; MCP server thường là nguồn lớn nhất)
- Mọi `CLAUDE.md` load bằng directory walk-up
- Mọi `.claude/rules/*.md` always-loaded
- Description của mọi model-invocable skill (chỉ metadata, không phải body)
- `MEMORY.md` (tối đa 25KB)
- Toàn bộ hội thoại trước đó: message của bạn, reply của Claude, và mọi tool result
- Nội dung file Claude đã đọc, grep output, bash output
- Task output Claude đang build ngay bây giờ

Một lần `Read` file source 2000 dòng có thể ăn 30–50k token. Mười lần đọc như vậy trong session debug dài và bạn đã đốt 5% window trước khi làm việc thật. Bảo Claude dump file 500 dòng ba lần — 150k token bay, cạnh tranh với description skill mà bạn cần Claude trigger đúng.

> **Đừng tin số trên — tự đo đi.** Claude Code có command `/context`, break down usage hiện tại của session theo category (system prompt, tool, CLAUDE.md, conversation, file, v.v.). Chạy nó trên vài session thật trước khi tối ưu bất cứ gì; tỷ lệ mix thay đổi rất khác nhau theo MCP setup, kích thước repo, và style hội thoại.

"1M token" là **trần**, không phải **không gian làm việc**. Không gian khả dụng thu hẹp mỗi phút session.

### Fact 2 — Mỗi request re-send toàn bộ hội thoại

LLM inference là **stateless**. Model không nhớ gì giữa các request. Mỗi lần Claude reply, client re-send toàn bộ hội thoại trước đó, mọi CLAUDE.md, mọi rule, mọi skill description, mọi tool definition — tất cả, từ turn 1.

Hệ quả thực tế: nếu bạn thêm 500 dòng instruction vào `CLAUDE.md`, 500 dòng đó được include trong mọi request cho đến hết session. "Upfront tax" không phải chi phí một lần — nó là chi phí mỗi turn nhân với mọi turn bạn sẽ có.

#### Nuance 1 — prompt caching làm giảm *chi phí*, không giảm *context tax*

API của Anthropic hỗ trợ [prompt caching](https://docs.claude.com/en/docs/build-with-claude/prompt-caching). Những điểm mà đa số guide bỏ qua:

- **Cache reads ~10% giá input.** Rẻ.
- **Cache writes tốn ~125% giá input.** Turn 1 với `CLAUDE.md` 1000 dòng *đắt hơn* không cache. Chỉ sau vài turn tiếp theo amortize được write cost thì discount mới bù lại.
- **TTL mặc định là 5 phút** (tier 1-hour có beta). Để session idle qua bữa trưa là cache evict — turn tiếp theo re-write và trả lại 125%.
- **Caching theo breakpoint.** API chấp nhận tối đa 4 marker `cache_control`. Claude Code đặt chúng ở đâu (system prompt, tool definition, CLAUDE.md, conversation) quyết định phần nào thực sự cache. Đụng vào bất cứ thứ gì trước một breakpoint thì mọi thứ từ breakpoint đó trở đi phải re-send.

Nên câu *"bạn trả mỗi turn"* đúng cho **context budget** (token vẫn chiếm window và vẫn cạnh tranh attention) nhưng phóng đại cho **chi phí dollar và latency** khi session đã warm. Ngược lại, nó *thiếu* cho session ngắn và sau mỗi idle gap.

Implication cho việc tối ưu `CLAUDE.md` và rules không đơn giản là *"giữ cho nhỏ."* Mà là:

- **Stable-within-session thắng small.** `CLAUDE.md` 1000 dòng không đổi qua session liên tục sẽ cache tốt sau turn trả write cost. Edit giữa session → cache invalidate → trả lại 125%.
- **Giữ volatile content ra khỏi upfront tax.** Nội dung thay đổi nhiều (debug state hôm nay, task note hiện tại) thuộc về hội thoại hoặc ephemeral memory, không phải `CLAUDE.md` nơi một lần edit phá cache cho mọi turn sau.
- **Session ngắn trừng phạt bloat.** Nếu đa số session của bạn <5 turn, cache-write cost thống trị và lời khuyên "stability" đảo chiều: small thắng stable-but-large.
- **Context budget vẫn finite.** Caching làm token rẻ hơn và nhanh hơn; không làm nó nhỏ hơn. Attention dilution (Fact 3 dưới) vẫn áp dụng dù token đến từ cache hay fresh.

#### Nuance 2 — `/compact` viết lại lịch sử hội thoại giữa session

Claude Code auto-compact khi context đầy. Sau compaction, client không resend hội thoại raw từ turn 1 — nó send **summary** cộng các turn gần nhất. Cụ thể:

- `CLAUDE.md` project-root **được re-inject** từ đĩa (instruction root-level survive).
- `CLAUDE.md` nested trong subdirectory **KHÔNG re-inject**; chỉ load lại khi Claude đụng file trong subtree đó.
- Skill body đã invoke **được re-attach** với token budget (gần nhất trước; invocation cũ có thể bị drop).
- Phần còn lại của hội thoại thành summary do model generate.

Nên *"mỗi request resend full conversation"* chỉ đúng đến compaction event đầu tiên. Session dài (>2 giờ) thường cross nhiều cái. Điều này quan trọng vì:

- Edit CLAUDE.md giữa session tệ hơn đã nói ở trên: nó không chỉ phá cache, mà còn là version được re-inject sau mọi compaction tiếp theo, có khả năng overwrite context Claude đã develop trước đó.
- Nested `CLAUDE.md` mong manh hơn root: một lần `/compact` và nó biến mất cho đến khi Claude đụng lại subdirectory đó.
- Skill invoke sớm có thể bị silently drop sau đủ compaction. Nếu skill critical cần persist, re-invoke.

### Fact 3 — Context lớn hơn không có nghĩa focus tốt hơn

Self-attention — cơ chế model dùng để quyết định token nào quan trọng — gán weight cho mỗi token, và **các weight này phải tổng bằng một giá trị cố định**. Điều đó có nghĩa mỗi token irrelevant thêm vào làm giảm weight available cho token relevant. Nhiều context làm signal loãng hơn, không sắc hơn.

Kết hợp với hiệu ứng "lost in the middle" được đo empirically (thông tin chôn giữa prefix dài và suffix dài bị weight ít hơn), một prompt gấp đôi có thể *kém hiệu quả hơn* prompt nửa kích thước, nếu phần thêm là noise. More context không phải là free upgrade.

Một hệ quả thực tế quan trọng đến mức cần tách ra: **số lượng skill (hoặc rule, hoặc MCP tool) tối ưu không phải "nhiều nhất có thể"; mà là ít nhất có thể trong ngân sách attention của target model của bạn.** Một setup chạy ngon lành trên Opus 4.x có thể gãy trên model coding 30B-parameter self-hosted không phải vì model nhỏ "dốt" — mà vì mỗi description, rule, tool schema thêm vào dilute weight attention available cho thứ Claude thật sự cần.

> **Deep dive:** [`docs/llm-mechanics/self-attention.vi.md`](./docs/llm-mechanics/self-attention.vi.md) — giải thích self-attention bằng analogy, vì sao 1M token là trần chứ không phải siêu năng lực, và ý nghĩa cho việc viết prompt Claude Code. Đọc nếu bạn chưa đọc transformer paper và muốn hiểu intuition mà không cần math.

### Ý nghĩa cho placement

Đa số core primitive guide này tập trung đều tồn tại để trả lời cùng một câu hỏi: *làm thế nào để feed model đúng thứ nó cần, đúng lúc nó cần, mà không trả phí trên mỗi turn ta không dùng?* (MCP server, plugin, và các ecosystem primitive khác tồn tại vì lý do khác — external integration, distribution — và được mention ngắn ở nơi khác.)

Ba category rộng, driven bởi cơ chế phía trên:

- **Trả mỗi turn**: `CLAUDE.md`, always-loaded rule, **skill description**. Bloat chúng và mọi response chậm hơn, đắt hơn, kém focus hơn.
- **Trả chỉ khi relevant**: **skill body**, path-scoped rule, memory topic file, subagent. Lối thoát cho content nặng.
- **Trả không gì cả ở mức model**: hook và `permissions.deny`. Enforcement sống ngoài context window hoàn toàn.

Đa số argument về placement trong phần còn lại của guide là hệ quả của ba fact trên cộng hai nuance. Câu hỏi thật không phải là *có nên* nghĩ theo category này không, mà là *knob nào* cá nhân bạn nên chạm vào cho codebase và session pattern của mình.

---

## Các primitive (tóm lược)

| Primitive | Bản chất | Lifetime | Scope | Kích hoạt bởi |
|---|---|---|---|---|
| `CLAUDE.md` | Behavioral guideline (prose) | Commit | Repo / dir | Claude đọc mỗi turn (consumption, không phải enforcement) |
| `.claude/rules/*.md` | Declarative invariant (MUST / MUST NOT) | Commit | Repo | Load cùng CLAUDE.md; variant path-scoped load khi đọc file khớp |
| `.claude/skills/*/SKILL.md` | Procedural workflow (how-to) | Commit | Repo / user | Claude invoke ngầm qua description match, hoặc user qua `/name` |
| `.claude/hooks/*.sh` | Machine enforcement | Commit | Repo | Harness ở pre/post tool lifecycle event (real enforcement) |
| `memory/*.md` | Ephemeral user / project state | User-local | Cross-session | `MEMORY.md` auto-load; topic file on demand |
| Subagent | Context isolation | Per-invocation | Task-scoped | `Agent` tool call bởi parent |

Mỗi primitive trả lời một câu hỏi khác nhau. Theo kinh nghiệm của tôi, lẫn lộn chúng là mis-placement thường gặp nhất.

> **Bảng này không phải toàn bộ set.** Những thứ guide này không cover sâu nhưng sống trong cùng ecosystem: **MCP server** (thường là nguồn upfront tool-schema token lớn nhất trong session), **plugin** (cơ chế chính thức để bundle skill + hook + agent), **settings** (`settings.json`, `settings.local.json`, managed setting — govern permission, hook, environment), **slash command** (merged vào skill cuối 2025; file ở `.claude/commands/x.md` và skill ở `.claude/skills/x/SKILL.md` đều tạo `/x`), và **output style**. Mọi thứ trong guide này áp dụng cho chúng, nhưng cover hết sẽ gấp đôi độ dài.

---

## Hai loại chi phí, một cách enforcement

Quyết định placement dễ hơn nhiều một khi bạn internalize rằng các primitive này sống trên **ba trục khác nhau**, không phải một. Rất nhiều docs đối xử với chúng như tool thay thế được cho nhau — không phải vậy.

### Trục 1 — Context tax mỗi turn

Những thứ này load vào context của Claude ở đầu mọi hội thoại, dù bạn có dùng hay không:

- `CLAUDE.md` — full content
- `.claude/rules/*.md` không có frontmatter `paths:` — full content
- **Skill description** (YAML frontmatter) — chỉ metadata, nhưng load cho mọi *model-invocable* skill (skill với `disable-model-invocation: true` excluded khỏi tax này — xem [Two triggers](#hai-trigger-explicit-vs-implicit) dưới)
- **Subagent description** — tương tự
- `MEMORY.md` — 200 dòng đầu hoặc 25KB, cái nào đến trước

Budget accordingly. Repo với 50 skill × 20 dòng description là ~1000 dòng upfront tax trước khi bạn gõ gì.

### Trục 2 — Prompt enrichment on-demand

Những thứ này chỉ vào context khi Claude cần. Nhưng "on-demand" không đồng nghĩa với "ephemeral" — *post-load lifecycle* của chúng khác nhau, và sự khác biệt quan trọng cho việc budget context:

| Primitive                         | Load khi                                    | Lifecycle sau khi load                                                                             |
| :-------------------------------- | :------------------------------------------ | :------------------------------------------------------------------------------------------------- |
| **Skill body**                    | Claude invoke skill                         | Vào transcript như một message; [docs xác nhận](https://code.claude.com/docs/en/skills#skill-content-lifecycle) nó stay hết session và được re-attach sau `/compact` với token budget |
| **Subagent execution context**    | Parent gọi tool `Agent`                     | Isolated hoàn toàn trong context window mới; parent chỉ thấy summary cuối                         |
| **Path-scoped rule** (`paths:`)   | Claude đọc file khớp pattern                | Inject khi file khớp được đọc; stay trong transcript như conversation history (không re-fetch ở turn sau) |
| **Memory topic file**             | Claude mở nó qua tool `Read`                | Hành xử như bất kỳ file read nào — content vào transcript và stay cho đến khi `/compact` summarise đi |
| **Nested `CLAUDE.md`**            | Claude đụng file trong subdirectory của nó  | Tích lũy khi bạn đi sâu hơn; **không re-inject sau `/compact`** (khác với project-root `CLAUDE.md`) |

Đây là nơi content nặng, task-specific thuộc về — free khỏi upfront "every turn" tax. Hai cái bẫy đáng nêu tên:

- **"On-demand" không phải "ephemeral."** Skill body và file read stay trong transcript sau khi load. Invoke 10 skill trong một session là tích lũy cả 10 body cho phần còn lại của session.
- **Lifecycle claim trong bảng này phản ánh behavior quan sát được và docs public tại thời điểm viết.** Cách Claude Code handle path-scoped rule re-injection và nested CLAUDE.md sau compaction đã evolve qua các release. Trước khi budget nặng quanh chúng, chạy thử nhỏ trên version hiện tại.

### Trục 3 — Hoàn toàn ngoài model

Hook khác về bản chất. Chúng chạy trong harness Claude Code (Node.js client), không trong model:

- Không tốn context token
- Deterministic: shell exit code quyết định block / pass; model không thể override
- Không thể bị "quên" giữa hội thoại
- Giới hạn ở những gì shell command thật sự có thể verify

Hook là một trong hai layer biến "MUST NOT" từ hy vọng thành enforcement. Layer kia là `permissions.deny` trong `settings.json`:

```json
{
  "permissions": {
    "deny": ["Bash(rm -rf *)", "Read(./.env)"]
  }
}
```

Cả hai chạy trong harness, cả hai deterministic, cả hai bypass model hoàn toàn. Hook linh hoạt hơn (shell logic tùy ý, có thể inspect tool input, có thể inject message back); `permissions.deny` đơn giản hơn và declarative. Dùng cả hai. Mọi primitive khác phụ thuộc vào việc Claude đọc, hiểu, và chọn follow.

---

### Trust-boundary split ở skill và subagent

Description và body sống trên các trục khác nhau — và chúng có thể nói những điều khác nhau.

|                 | Claude thấy khi nào        | Dùng để làm gì                      |
| --------------- | -------------------------- | ----------------------------------- |
| **Description** | Mỗi turn (Trục 1)          | Routing: *"có nên invoke không?"*   |
| **Body**        | Chỉ sau khi invoke (Trục 2) | Execution: *"bây giờ tôi làm gì?"* |

Claude route dựa trên description một mình. Body không có cơ hội "cãi lại" cho đến sau khi quyết định invoke đã được đưa ra. Split này vừa là design feature vừa là footgun:

- **Feature.** 50 skill không tốn 50 × body-length token mỗi turn. Chỉ description của chúng thôi. Đó là cái làm thư viện skill lớn khả thi.
- **Drift footgun.** Description hứa X nhưng body làm Y khiến Claude invoke trong context sai — và bạn chỉ phát hiện mismatch sau khi skill đã chạy xong.
- **Supply-chain risk — có thật, nhưng không phải không có phòng thủ.** Skill share từ repo ngoài có thể pair description lành tính ("code formatter") với body không mong đợi. Defense tồn tại theo layer: Claude Code prompt approval trên external import, PreToolUse hook fire trên tool call bên trong body, `permissions.deny` block tool không cho phép, và user thấy mỗi tool call trong UI trước approval (ở default mode). Threat thực tế = cần review kỹ cho skill bên thứ ba, không phải mỗi lần cài đều bị auto-pwn. Nhưng trust-boundary split vẫn có thật: description bạn scan không phải thay thế cho body bạn scan.

> **Description là routing contract. Body là execution contract. Chúng có thể phân kỳ — và đó vừa là feature vừa là vulnerability. Khi adopt third-party skill, đọc body, không chỉ description.**

---

### Hai trigger: explicit vs implicit

Skill và subagent có thể được invoke theo hai cách khác nhau, và lựa chọn này ảnh hưởng đến reliability, safety, và context cost. Làm đúng cái này thường quan trọng hơn nội dung của bất kỳ skill cụ thể nào.

Có hai cách invoke skill hoặc subagent:

1. **Explicitly** — user gõ `/skill-name`, hoặc parent agent gọi tool `Agent` với subagent type cụ thể. Deterministic. User (hoặc parent model) kiểm soát timing.

2. **Implicitly** — Claude đọc description mỗi turn, match với prompt hiện tại, và tự quyết có invoke hay không. Đây là mode "nó tự biết tôi muốn gì". Cũng là mode mà nhiều hành vi trông giống thông minh thật ra là careful description engineering.

Claude Code expose knob explicit cho cả hai trong skill frontmatter:

| Frontmatter                         | Invocation                                        | Dùng cho                                                                                                                                  |
| :---------------------------------- | :------------------------------------------------ | :----------------------------------------------------------------------------------------------------------------------------------------- |
| (default)                           | Cả `/command` và auto-trigger                     | Đa số skill                                                                                                                                |
| `disable-model-invocation: true`    | Chỉ `/command`. Claude không thể auto-trigger.    | Bất cứ thứ gì có side effect: `/commit`, `/deploy`, `/send-slack-message`. Bạn không muốn Claude tự quyết *khi nào* deploy dựa trên vibe. |
| `user-invocable: false`             | Chỉ Claude auto-trigger. Ẩn khỏi menu `/`.        | Background knowledge (`legacy-system-context`, `api-conventions`) — không actionable như command, nhưng Claude nên pull vào khi relevant. |
| `paths: src/billing/**`             | Claude chỉ auto-trigger khi đụng file khớp        | Skill domain-specific không nên cạnh tranh attention ngoài phạm vi của nó                                                                  |

Subagent có split tương tự: parent model pick subagent type nào spawn dựa trên description tool `Agent`, nhưng bạn cũng có thể constrain agent nào available trong context nhất định.

**Vì sao điều này quan trọng:**

- **Implicit trigger phụ thuộc vào description engineering.** Khi một skill như "tự biết" khi nào kích hoạt, là vì có người đã tune description, `when_to_use` hint, và path scope để model reliably auto-invoke trên prompt đúng. Đó là công việc thật — và failure-prone. Một description phải thắng attention competition chống lại mọi skill khác trong budget.
- **Explicit trigger là đòn bẩy an toàn, đặc biệt trên model yếu.** Auto-trigger giả định model đủ thông minh để match prompt với description đúng. Frontier model (Opus 4.x, Sonnet 4.6, và Haiku 4.5 trong thực tế) auto-trigger đủ tốt cho đa số skill description viết tốt. Gap hiện rõ trên model coding open-source nhỏ hơn trong 30B-parameter class — DeepSeek Coder, Qwen Coder, CodeLlama và tương tự. Model này ngày càng capable *execute* skill body sau khi invoke, nhưng *routing* accuracy (match user prompt với description skill đúng, trong 30+ candidate, mỗi turn) giảm thấy rõ. Skill không liên quan fire, skill liên quan bị skip. Nếu bạn target OSS model cho setup self-hosted, forcing `/command` invocation là cách rẻ nhất để tránh cái này.
- **Action có side effect nên gần như luôn explicit.** Nếu skill deploy, send message, mutate shared state, hoặc spend money, bạn không muốn "Claude đã quyết định" trong post-mortem. `disable-model-invocation: true` là cheap insurance.
- **Pure knowledge injection thì implicit ổn.** Skill nói *"khi đụng billing code, đây là invariant"* là auto-trigger material lý tưởng — scope với `paths:`, ẩn khỏi menu với `user-invocable: false`. User không nên phải nhớ `/billing-context`.

---

## Appendix — decision tree: "knowledge này sống ở đâu?"

Mental model phía trên là sản phẩm thật của guide này. Tree này là appendix áp dụng model vào câu hỏi placement thường gặp nhất. Dùng nó để pick **primary placement**, sau đó xem note supplementary-placement dưới — knowledge thật thường thuộc về nhiều hơn một primitive.

```
Nó có thể derive từ `git log` hoặc đọc code với chi phí thấp không?
  └── CÓ → không lưu gì. Đừng ô nhiễm context.

Nó là MUST / MUST NOT luôn đúng, bất kể task?
  ├── Machine-checkable (grep, lint)? → rule + hook (cả hai, không phải chọn một)
  └── Human-enforced only?            → rule

Nó là workflow "khi user nói X, làm Y₁ → Y₂ → Y₃"?
  └── CÓ → skill

Trả lời nó có cần đọc đủ source để noticeably ô nhiễm main context không
(rule of thumb: vài nghìn dòng read)?
  └── CÓ → subagent (isolate khỏi main context)

Nó là fact về user hoặc project thay đổi theo thời gian?
  └── CÓ → memory

Nó là style / tone / philosophy trải khắp repo?
  └── CÓ → CLAUDE.md
```

**Supplementary placement.** Cùng một mảnh knowledge thường xứng đáng có home trong nhiều hơn một primitive — không phải vì bạn đang duplicate, mà vì mỗi primitive address một failure mode khác:

- *"Mọi outbound HTTP phải dùng wrapper `httpclient.Do` của chúng tôi"* → **rule** (declarative invariant Claude đọc) + **hook** (grep deny `http.Get(` raw trong PreToolUse) + optional **skill** (`/migrate-http-call` để refactor call site hiện tại).
- *"Billing code invariant"* → path-scoped **rule** (chỉ load khi đụng `src/billing/**`) + optional **skill** (`/billing-review`, `user-invocable: false` để Claude auto-load trong billing context).
- *"Style commit message ưa thích"* → **CLAUDE.md** (convention một dòng) + explicit **skill** `/commit` (procedure cho commit scripted).

Rule of thumb: nếu cái gì là *declarative + enforceable*, làm cả rule và hook. Nếu cái gì là *knowledge + procedure*, xem xét cả rule/CLAUDE.md và skill. Decision tree pick placement **quan trọng nhất**; nó không nói "chỉ đặt ở đó".

Nếu không có cái nào match, bạn có lẽ không cần persist nó.

---

## Bắt đầu nhỏ, curate có chủ đích

Hai ý đáng mang theo khi đọc xong.

**1. Tổ chức vì weakness của model, không phải làm ngơ nó.**
Một `.claude/` được curate tốt không chỉ tiện — nó là *chiến lược quản lý context*. Mỗi item bạn thêm hoặc giúp model focus, hoặc chống lại nó. Mechanics sớm hơn trong guide này không phải physics trừu tượng; chúng là ràng buộc ngân sách mà curation của bạn hoạt động dưới đó. Dùng chúng làm checklist:

- **Ít description skill hơn, sắc hơn** → ít attention dilution (Fact 3). Skill target thắng Q/K match thường xuyên hơn. Mười description tune tốt thường thắng ba mươi description generic.
- **Path-scoped rule thay vì always-loaded** (`paths: src/billing/**`) → rule vào context chỉ khi relevant, giải phóng upfront budget cho thứ model cần globally.
- **Subagent cho heavy read** → 50k token file content sẽ polluted main session nay stay trong window isolated, và bạn nhận về summary thay vì raw dump.
- **Hook cho MUST rule** → zero context cost, zero phụ thuộc vào Claude nhớ qua turn, và zero attention budget spend cho enforcement.
- **`permissions.deny` cho hard block** → deal tương tự hook cho lệnh cấm không cần logic.
- **CLAUDE.md stable trong session** → cache giữ warm, cost và latency giảm (Caching nuance).
- **Prune MCP server bạn thật sự không dùng** → thường là token win đơn lẻ lớn nhất. 5–10k token về lại mỗi turn, phần còn lại của session.

Câu hỏi để hỏi mỗi item trong `.claude/` của bạn: *nó tốn gì cho tôi về token, và tôi mất gì nếu không có nó?* Nếu bạn không thể trả lời nửa sau, nó là dead weight. Delete.

**2. Bắt đầu nhỏ. Cherry-pick trước khi tự viết.**
Bạn không cần tên, repo, hay master plan. Bạn chỉ cần một thư mục `.claude/` với một vài skill, rule, hook, và dòng CLAUDE.md fit codebase, model, và session pattern của bạn.

Cách tốt nhất để build cái đó không phải là viết từ scratch. Là lắp ráp:

- Browse skill và agent do người khác share (bundle phổ biến, blog post, setup của đồng nghiệp). Nhấc những cái giải quyết vấn đề bạn thật sự có, adapt theo naming và convention của bạn, bỏ cái không fit. Bạn own bản copy của mình từ lúc đó.
- Thêm từng cái một khi nhu cầu xuất hiện, không phải upfront:
  - Thêm một **rule** cho một invariant bạn hiện đang catch trong code review.
  - Thêm một **skill** cho một procedure bạn gõ cùng cách mỗi lần (`/commit`, `/review-pr`, gì cũng được).
  - Thêm một **hook** khi bạn nhận ra Claude cứ vi phạm cùng một rule và bạn mệt mỏi với việc nhắc nhở nó.
- Mỗi lần thêm cái gì đó, chạy `/context` và hỏi: *có đáng với chi phí không?*

Sau sáu tháng bạn sẽ có một setup không ai có — và không ai cần. Đó là mục tiêu: không phải framework bạn publish, mà là môi trường làm việc fit *codebase của bạn* và *ngân sách attention của model*. **Bạn đã biết codebase của mình kỹ hơn bất kỳ config author nào trên GitHub. Mindset phía trên là cái cho phép bạn hành động trên kiến thức đó.**

---

## Scope boundaries

- Không phải danh sách "awesome-claude". Đã có repo khác.
- Không phải tutorial cài Claude Code. Xem [official docs](https://docs.claude.com/en/docs/claude-code).
- Không phải handbook toàn diện về prompt engineering. Nó chạm vào các chủ đề prompt-engineering-adjacent (attention dilution, description craft, caching) ở chỗ chúng load-bearing cho argument; general prompt craft ngoài scope.
- Không phải ranking các Claude model. Behavior model cụ thể mention ở chỗ chúng shift argument, nhưng volatile đủ để bạn nên tự re-verify thay vì tin snapshot của guide này.

Issue và disagreement được welcome. Nếu bạn nghĩ một framework implication ở đây sai, mở issue với counter-example cụ thể — guide này hữu ích hơn khi là claim bị tranh luận hơn là doctrine không bị thách thức.

## License

MIT — xem [LICENSE](./LICENSE).
