# The delegation ceiling

*[English] · [Tiếng Việt](./delegation-ceiling.vi.md)*

*Optional companion reading to the main README. What magical delegation actually looks like when it works — and why the underlying problem does not go away even as models, context windows, and infrastructure keep improving.*

The main README's opening section takes for granted a few claims: that the "delegate, don't dictate" advice genuinely works at the single-prompt level, that the model's trajectory is real, and that even so, there is a floor the trajectory does not cross. Readers who are already convinced can skip this file. Readers who would like one concrete story and a deeper reality-check should read on.

---

## Smart reasoning is not a substitute for durable knowledge

When the loop tightens, the result can be genuinely magical. A real example:

I once applied a Terraform config to create a server. It failed. Without any hand-holding, Claude:

1. Inferred the URL might be the problem,
2. Went to look at the provider's source to verify — but the provider was named against convention in my `.tf` file, so Claude could not find it locally,
3. Web-searched the closest matching name,
4. Found the actual GitHub repo, read the source,
5. Discovered my config URL had a trailing `/` that should not be there. Fixed it.

Mind-blowing, honestly. And of course, after that I did not dare close the session — surely that expertise lived *somewhere* in there. Sure enough, N turns of unrelated work later, a similar issue cropped up and Claude had to re-derive half of it from scratch.

**The lesson:** reasoning is not the same as durable knowledge. A model smart enough to *derive* the right answer once will still re-derive it, imperfectly, every time its context drifts or resets. In a long session, attention rot eventually buries the answer. In a fresh session, it is never there to begin with. You need a way to make the conclusion *stick.*

---

## The trajectory genuinely helps

Models keep getting smarter. Context windows keep growing — 8k → 200k → 1M → whatever comes next. The infrastructure around the model keeps improving too — smarter caches, better compression of old conversation, cleaner ways to isolate heavy work. It is fair to assume the experience gets a little more forgiving every year, and that a lot of the rough edges people hit today will soften on their own.

---

## But the trajectory has a floor

As long as the underlying system is an **LLM driven by self-attention**, three things remain true no matter how large the window gets:

1. The window is still finite.
2. Every token still competes for a fixed attention budget (so more context means thinner signal per token — the "lost-in-the-middle" effect does not disappear at 10M tokens any more than it did at 8k).
3. Inference is still stateless — the full conversation re-sends on every turn.

Engineering improvements push the ceiling up; they do not change the shape of the ceiling. Compaction buys you a larger effective session, but it buys it by *discarding content*, not by remembering more. Prompt caching makes re-sends cheap, but the re-send still has to fit. **The problem scales with your ambition, not with the model** — the day a 10M-token window arrives, you will find yourself wanting to pack 20M tokens of project context into it.

---

## Related deep dives

- [`why-context-matters.md`](./why-context-matters.md) — the three facts about LLM runtime in more detail, with examples of what each one costs you in practice.
- [`self-attention.md`](./self-attention.md) — why the "lost-in-the-middle" effect is mechanical, not a bug a bigger window fixes.
- [`per-turn-cost-math.md`](./per-turn-cost-math.md) — what caching, compaction, and a mid-session `CLAUDE.md` edit actually cost.
