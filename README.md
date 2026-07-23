# AI-Powered Competitor News Monitor — Weekly Digest

An n8n workflow that monitors competitor blogs on a weekly schedule, uses an LLM to extract competitive takeaways, and delivers a formatted digest by email.

Built on n8n Cloud. Tested end to end — a sample of the delivered output is included below.

---

## Architecture

```
Schedule Trigger (Mon 9AM)
      │
      ├──→ HTTP Request: Zapier RSS  ──┐
      │                               ├──→ Merge ──→ Code: Parse / Filter / Dedupe
      └──→ HTTP Request: n8n RSS    ──┘                        │
                                                               ▼
                                              AI Summarize (Groq · Llama 3.3 70B)
                                                               │
                                                               ▼
                                                Code: Format Digest (Markdown)
                                                               │
                                                               ▼
                                                        Send Email
```

| Node | Role |
|---|---|
| **Weekly Trigger — Mon 9AM** | Schedule Trigger, weekly cadence |
| **Fetch Zapier Blog RSS** | HTTP Request → `zapier.com/blog/feeds/latest/` |
| **Fetch n8n Blog RSS** | HTTP Request → `blog.n8n.io/rss` |
| **Merge Feeds** | Append mode, fans two branches back into one stream |
| **Parse, Filter & Dedupe** | Code node — XML parsing, normalization, 7-day filter, dedupe, per-source cap |
| **AI Summarize** | Basic LLM Chain → structured JSON takeaways |
| **Groq (OpenAI-compatible)** | Model sub-node, `llama-3.3-70b-versatile`, temp 0.3 |
| **Format Digest** | Code node — validation + Markdown assembly |
| **Send Digest** | Email delivery |

---

## Design decisions

### Parallel fetch, not sequential

Both HTTP nodes branch directly off the trigger rather than chaining. Three reasons:

- **Concurrency** — feeds fetch simultaneously. Marginal at two sources, meaningful at ten.
- **Failure isolation** — in a chain, a hung first request blocks the second. In parallel branches, one dead feed doesn't prevent the other from arriving.
- **Extensibility** — adding a third source is one more branch, not surgery on a chain.

The tradeoff, stated honestly: this is less DRY than holding feed URLs in an array and looping. The array approach scales better; the parallel approach is more legible on the canvas and lets each feed be configured independently. For a build meant to be read, legibility won.

### Graceful degradation on feed failure

Both HTTP nodes are set to **Continue (using error output)**. A weekly digest that dies entirely because one feed 404s is worse than a digest built from the surviving source. The parser downstream guards against non-string and empty input, so a failed branch contributes nothing rather than throwing.

This was validated accidentally during development: the initial n8n feed URL returned 404 while Zapier succeeded, and the workflow continued rather than failing.

### Normalize heterogeneous sources early

The two feeds do not share a schema:

| | Zapier | n8n |
|---|---|---|
| Text encoding | plain | `<![CDATA[...]]>` wrapped |
| Author field | `<author>` | `<dc:creator>` |
| Full body | absent | `<content:encoded>` (full article HTML) |

The parser strips CDATA, removes HTML tags, and decodes entities so both produce an identical internal shape: `{source, title, link, summary, published}`. Everything downstream — prompt, formatter, output — works against one schema regardless of how many feeds are added.

`content:encoded` is deliberately **not** extracted. Pulling full article HTML would multiply token count for no analytical gain; RSS descriptions already carry the lede.

### Filter before inference, not after

The 7-day window, deduplication, per-source cap, and a 400-character summary truncation all run **before** the LLM call. Roughly 80 KB of raw XML is reduced to ~3 KB of normalized text.

Fetching is free; inference is not. Any work that can be done deterministically should be done deterministically, and done first.

### Structured output over prose

The prompt requests strict JSON with a defined schema rather than free-form summary text. This makes the formatter deterministic — it reads named fields instead of regex-parsing prose that changes shape between runs.

Temperature is set to **0.3**: this is summarization, not creative writing. The same input should produce substantially the same digest, and the model should stay anchored to source text.

### Recipient address is a placeholder in the committed workflow; it is a per-deployment value.

---

## Bugs found and fixed

Both of these surfaced from running the workflow against real data rather than from reasoning about it in advance.

### 1. Source starvation from a global item cap

**Symptom.** The digest contained only Zapier articles. `sourcesFound` reported one source despite both feeds succeeding.

**Cause.** The original logic sorted all articles by recency and truncated to a global cap of 12. Zapier publishes ~18 articles/week; n8n publishes ~1. Every Zapier article was newer than n8n's newest, so the cap consumed all 12 slots before n8n was ever reached. The lower-cadence source was structurally guaranteed to lose.

For a competitor-monitoring tool, silently dropping an entire competitor is the worst available failure mode — the absence is indistinguishable from "they published nothing."

**Fix.** Replaced the global cap with a **per-source quota** (`PER_SOURCE_CAP = 6`). Each feed is ranked against itself and guaranteed representation. Worst-case token spend is now `PER_SOURCE_CAP × number_of_feeds`, which is a predictable knob as sources are added.

A `perSourceCounts` field was added to the node output, exposing how many articles each feed actually published. This is what makes cadence gaps visible instead of silent.

### 2. LLM hallucinated its own coverage

**Symptom.** The model returned `"sources_covered": ["Zapier Blog", "n8n Blog"]` while every takeaway cited Zapier. It claimed coverage of a source it never used.

**Cause.** `sources_covered` was delegated to the model despite being fully derivable from the takeaways it had just written.

**Fix — two layers:**

1. **Prompt constraint.** Added an explicit rule that `sources_covered` may list only sources appearing in takeaways, plus a rule requiring a brief low-signal takeaway for any source that published but produced nothing significant — so a quiet competitor is reported as quiet rather than omitted.

2. **Deterministic recomputation.** The formatter ignores the model's `sources_covered` entirely and rebuilds it from the takeaway array. Model-reported coverage is never trusted.

**Principle:** anything an LLM asserts that can be computed deterministically should be computed deterministically. Asking a model for a derivable value is inviting error for no benefit.

---

## Reliability measures

- **Defensive JSON parsing.** The formatter strips stray markdown fences and wraps parsing in try/catch.
- **Degraded-mode fallback.** If the LLM returns unusable output, the workflow still ships a digest — containing the raw article list, subject-line-tagged `[Degraded]`. A monitoring workflow that fails silently is worse than one that fails loudly.
- **Coverage footer.** Every digest reports how many sources were scanned and how many articles each published, flagging any source that had articles but produced no takeaway. The digest is auditable against its own inputs.

---

## Sample output

```
# Competitor News Digest
**Week ending Thu, 23 Jul 2026**

### Zapier emphasizes determinism and enterprise capabilities

## Key Takeaways

**1. Zapier Pushes Determinism**

Zapier is promoting deterministic automation as a cost-effective alternative to
agent-based automation, highlighting the importance of predictable outputs in
certain workflows. This approach may help Zapier differentiate itself from
competitors that rely heavily on AI agents.
[Read more](https://zapier.com/blog/ai-by-zapier-guide) · _Zapier Blog_

**2. Zapier Targets Enterprise**

Zapier is actively marketing its platform as an enterprise-ready solution,
emphasizing ease of use and scalability. This strategic positioning suggests
Zapier is seeking to expand its customer base to include larger organizations.
[Read more](https://zapier.com/blog/zapier-for-enterprise-automation) · _Zapier Blog_

**3. n8n Discusses LLM Production**

n8n has published an article discussing the differences between fine-tuning and
RAG approaches for production LLM systems. While not directly competitive, this
demonstrates n8n's expertise in AI and automation.
[Read more](https://blog.n8n.io/fine-tuning-vs-rag/) · _n8n Blog_

---
## Coverage

Scanned **2** sources over the last **7** days.

- **Zapier Blog**: 16 articles published
- **n8n Blog**: 1 article published

_7 articles analyzed. Generated automatically by n8n._
```

---

## Known limitations

- **Plain-text delivery.** Markdown arrives unrendered — literal `#` and `**` are visible. Rendering would require a Markdown→HTML conversion step. Accepted as a conscious tradeoff; the brief permits plain text or Markdown.
- **Deactivated.** The workflow is schedule-ready but left inactive; it was tested via manual execution.

---

## Running this yourself

1. Import `workflow.json` into any n8n instance
2. Add an **OpenAI-type credential** with base URL `https://api.groq.com/openai/v1` and a Groq API key (free tier at [console.groq.com](https://console.groq.com))
3. Add an email credential and set the recipient in the final node
4. **Execute Workflow** to test, or activate for the Monday 9 AM schedule

Groq is used through n8n's OpenAI node because Groq's API is OpenAI-compatible — only the base URL differs. This satisfies the "AI / OpenAI node" requirement at zero cost, and the model is swappable without touching the prompt or any downstream node.