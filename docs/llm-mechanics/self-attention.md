# Self-attention, in plain words

*[English] · [Tiếng Việt](./self-attention.vi.md)*

*Why the 1M-token context window is a ceiling, not a superpower.*

This doc is for people who use Claude Code but have not read transformer papers. No math. Analogies where possible. If you want rigor, the papers linked at the bottom have it.

---

## What is a token?

A token is the unit the model reads. Roughly a word or a chunk of a word.

- `"Hello world!"` → about 3 tokens
- A typical function definition → 50–200 tokens
- A 2000-line Go file → 30–50k tokens

Everything the model sees — your messages, its own replies, CLAUDE.md, tool definitions, file reads — is converted into a sequence of tokens and concatenated into one big sequence before being fed in.

---

## What is "attention," intuitively?

Read this sentence:

> *"The cat sat on the mat because **it** was tired."*

When your brain hits **"it,"** it automatically jumps back and links to **"cat,"** not "mat." That reach-back-and-link is what attention does for the model.

When an LLM generates the next token, it looks across every token already in the window and decides: *how much does each one matter to what I am writing right now?* Each token gets a weight. The weighted combination shapes the next output.

This happens not once, but for every token the model produces. And not on one "relevance channel," but on many in parallel — different attention heads specialize in different relationships (syntax, reference, style, etc.).

This mechanic is not Claude-specific. It is core to every modern LLM: GPT, Gemini, Llama, Mistral, DeepSeek, Claude — all of them are transformers, all of them are bound by it. The practical rules at the bottom of this page apply regardless of which model you are using.

---

## But how does it actually pick "cat" and not "mat"?

Fair question. Both are nouns. Both are grammatical candidates. The model does not roll a die. Two ingredients work together: **what the model learned during training**, and **how attention applies that knowledge at inference time**.

### Training: where the priors come from

During training, the model saw hundreds of billions of sentences. In those sentences:

- Pronouns like "it" almost always refer back to a noun that appeared recently — so the model learned a bias toward recent noun-like tokens.
- Certain nouns co-occur much more often with certain predicates. `"cat" + "tired"` appears far more often than `"mat" + "tired"` in natural text, because mats do not get tired and cats do.
- Subjects are more often the antecedents of pronouns than objects.

The model does not memorize rules explicitly ("if pronoun, link to subject"). It internalizes them as billions of numerical weights inside its neural network — weights that make certain linkages statistically more likely than others on any new input.

Without training, this capability simply does not exist. A freshly initialized model would link "it" to "cat" and "mat" with equal probability. **Training is where the knowledge lives.** Attention is the mechanism that retrieves the right piece of it for a specific input.

### Attention at inference: Q, K, V

When the model processes an input, every token produces three vectors (short lists of numbers) that encode different facets of what it is and what it needs:

- **Query (Q)** — "what am I looking for in the prior context?"
- **Key (K)** — "what do I advertise about myself, for others to find me?"
- **Value (V)** — "if I am selected as relevant, what do I contribute?"

These three vectors are not written by hand. They are produced by small learned transformations (themselves shaped by training). Every token has its own Q, K, and V.

When the model gets to **"it"**, it produces a Query that effectively says: *"I am a pronoun. I am looking for a recent, singular, animate-ish noun that can be the subject of 'was tired'."* Every prior token has already published a Key:

- `cat` — *"singular animate noun, subject of the current clause"*
- `mat` — *"singular inanimate noun, object of a preposition"*
- `sat` — *"past-tense verb, predicate"*
- ...

The model computes similarity between "it"'s Query and every prior token's Key. `cat`'s Key matches most closely, so `cat` gets the highest attention weight. The Value vectors are then combined, weighted by those similarities, and the result is what "it" carries forward into the next layer of computation.

A working analogy: think of it as a **fuzzy library catalog search**. Every prior token has placed a card in the catalog (its Key). The current token arrives with a search question (its Query). The model does a fuzzy similarity match and ranks all cards. The winning card's content (its Value) is pulled out and mixed into the answer.

### So: training or attention?

**Both, at different times.**

- *Training* decides what patterns the model knows at all. A model never trained on English cannot resolve English pronouns no matter how clever its attention.
- *Attention* decides which of those patterns fire on today's specific input. It is the retrieval mechanism that picks what matters here, now.

You will occasionally read "attention is how LLMs think." That is compressing two separate things together. More precisely: attention is **how the model applies, at inference, the knowledge that training baked in.**

### Why this matters for your prompts

- **Relevance is computed, not declared.** You cannot just write *"pay attention to X"* and expect the model to obey. The Q/K match still runs against everything in context. Your instruction helps only if it reshapes the actual attention math.
- **Noise competes for attention weight.** Paste a 500-line log alongside your question, and tokens from that log carry real weight in the computation. They do not politely wait to be ignored.
- **Training sets priors.** A model trained heavily on Python has stronger attention patterns for Python than for, say, Erlang. That is why Claude is noticeably better at some languages, frameworks, and tools than others — not magic, just training distribution.

---

## Why is it called "self"-attention?

Because every token attends to every other token in the same sequence. It's a pairwise relationship between all tokens.

For a sequence of N tokens, the model computes roughly N × N attention weights:

| Tokens in window | Pairwise weights computed |
| ---------------- | ------------------------: |
| 1,000            | 1,000,000                 |
| 10,000           | 100,000,000               |
| 100,000          | 10,000,000,000            |
| 1,000,000        | 1,000,000,000,000         |

Modern inference uses engineering tricks to keep this tractable, but the fundamental shape is O(N²): doubling the context roughly quadruples the work.

That is one reason why a 1M-token context is not "free" even on the vendor's side — it is a genuine technical feat to make it run at all.

---

## The catch: attention is a zero-sum pie

Here is the part that changes how you think about prompts.

The attention weights a token distributes across all prior tokens **must sum to 1**. It's a softmax — a fixed pie, divided up.

Imagine a spotlight. The total wattage is fixed. If you point it at:

- **10 items** → each gets 10% of the light
- **1,000 items** → each gets 0.1%
- **1,000,000 items** → each gets 0.0001%

The model can still "see" everything in the window. But every additional irrelevant token **reduces the weight available for relevant tokens**. In long contexts, important signals drown in noise.

This is the mechanical reason behind a counterintuitive fact: *a prompt twice as long can produce a worse answer than a prompt half the size, if the extra content is noise.* It is not that the model "forgets" — it just has less attention-weight to spare for what actually matters.

---

## The "lost in the middle" effect

Empirically, models do best on information near the **start** and **end** of their context, and worst on info in the **middle**. This has been measured across many models ([Liu et al. 2023](https://arxiv.org/abs/2307.03172)).

Concretely: if your 10,000-line CLAUDE.md has a critical rule at line 5,000, it is closer to useless than useful. The model has it in context, but attention cannot reliably find it.

This is why the Claude Code docs recommend keeping CLAUDE.md under ~200 lines. It is not an arbitrary style preference — it is working around a known weakness of the architecture.

---

## So why does Claude Opus advertise 1M tokens?

Because architecturally the model *can* attend across 1M tokens. For rare tasks — whole-repo refactors, analysing long legal documents, synthesising across many files — this is genuinely valuable.

But:

- **Can ≠ should.** Using the full window typically degrades answer quality due to attention dilution.
- **1M is the ceiling, not the workspace.** Your effective working set is much smaller.
- **Bigger context is a reserve, not a perf upgrade.** Treat it like RAM: having 64GB does not mean you should hold every file open at once.

If you are paying extra for a long-context model and using it the same way you used an 8k-context one, you are paying for capacity you are not benefiting from.

---

## Practical rules for Claude Code

Everything below follows from the three mechanics above (pairwise attention, zero-sum weights, lost-in-the-middle):

1. **Shorter prompts often outperform longer ones.** If you can say it in 50 lines instead of 500, do.
2. **Put critical rules near the top** of CLAUDE.md, not buried in the middle.
3. **Use path-scoped rules** (`paths:` frontmatter in `.claude/rules/`) — only load when Claude touches matching files. Less dilution for the rest of the session.
4. **Use skills** for content that only matters when invoked. Their bodies do not dilute attention until Claude actually invokes them.
5. **Use subagents for heavy reads.** A subagent has its own fresh window — the parent conversation is not polluted by its research.
6. **Watch for context rot.** Long sessions accumulate tool output, file reads, and compacted summaries. At some point, starting fresh is faster than continuing.

---

## Want to go deeper?

- [**"Lost in the Middle: How Language Models Use Long Contexts"** — Liu et al. 2023](https://arxiv.org/abs/2307.03172) — the empirical study behind the effect
- [**"Attention Is All You Need"** — Vaswani et al. 2017](https://arxiv.org/abs/1706.03762) — the original transformer paper (technical)
- [**Claude Code: context window**](https://code.claude.com/docs/en/context-window) — how Claude Code budgets and visualizes the window in practice
