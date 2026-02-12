# EN → AR Translation Pipeline
## Project Index & Overview

*Version 1.0 · Last updated: 2026-02-12*

---

## What Is This Project?

This pipeline takes an English static website and produces a fully functional Arabic version — same layout, same interactive elements, same links — just in Arabic and reading right-to-left.

It is not a simple "paste into Google Translate" process. The system is designed to handle the hard parts that break naive translation:

- **Code snippets and CLI commands** that must never be touched
- **Terminology consistency** — the same technical term must always translate the same way
- **HTML structure fidelity** — lists, links, and formatting must survive translation intact
- **Quality verification** — a mathematical check that the Arabic actually means what the English said
- **Human review** — a fallback for the cases the machine isn't confident about

The entire pipeline is **stateless**: each stage stores its results in a database, so you can extract pages on Monday, translate on Tuesday, and publish on Wednesday without keeping anything in memory.

---

## How The Pipeline Works — Plain English

Think of it as an assembly line with eight stations:

> **Extract** the text → **Protect** the technical bits → **Mine** the glossary → **Translate** → **Validate** the output → **Verify** the meaning → **Review** edge cases → **Restore** into final HTML

Each station hands off a database record to the next. If anything fails at any station, it goes back — not forward.

---

## Phase Overview Table

| # | Phase | What It Does | Why It Exists | How It Works | Key Tools |
|---|-------|-------------|---------------|-------------|-----------|
| **A** | [Extract & Map](/phase-a-extract-map.md) | Pulls all translatable text out of HTML and assigns each piece a stable address (PathID) | So we can put the Arabic back in exactly the right place, even days later | AngleSharp crawls the DOM; each node gets a coordinate like `0,1,5,2`; text is converted to Markdown | AngleSharp, ReverseMarkdown |
| **B** | [Protect & Mask](./phase-b-protect-mask.md) | Wraps code, CLI commands, paths, and brand names in placeholder tokens | The LLM must never alter `sudo apt-get` or `/var/www` — masking makes them invisible to translation | Regex + dictionary scan replaces protected content with `[[M_0]]`, `[[M_1]]`… stored in a manifest | Compiled Regex, FlashText |
| **C** | [Glossary Mining](./phase-c-glossary-mining.md) | Analyses the entire site before translation to discover important technical terms | Ensures "dependency injection" is always translated the same way across all 500 pages | TF-IDF, NER, co-occurrence analysis, link-graph ranking → top terms sent for human approval | spaCy, GLiNER, gensim, NLTK |
| **C2** | [Context Creation & Embedding](./phase-c2-context-embedding.md) | Pre-assembles the full LLM prompt payload for every unit, pre-computes English embedding vectors, and runs Translation Memory lookups | So Phase D is pure throughput — no inline context building, glossary filtering, or TM lookups on every call (including retries) | MiniLM encodes all English units in one batch; TM similarity search flags exact duplicates to skip translation entirely; ContextPacket stores section title, preceding paragraph, filtered glossary, content type, and TM candidates | MiniLM (ONNX), FAISS, gensim |
| **D** | [Translate](./phase-d-translate.md) | Sends masked text to the LLM along with context, glossary, and style rules | LLMs without context drift, hallucinate, or ignore glossary — structured prompts prevent all three | Each request includes: masked text + previous paragraph + section title + glossary + style guide | LLM (Gemini), Translation Orchestrator |
| **E** | [Validate](./phase-e-validate.md) | Runs automated checks on the Arabic output before it is saved | Catches structural breakage (missing list items, corrupted placeholders) before it reaches the DB | Rule engine checks placeholder parity, Markdown structure, glossary usage | Custom rule engine |
| **F** | [Semantic Gate](./phase-f-semantic-gate.md) | Mathematically compares English and Arabic meaning using AI embeddings | Detects when the LLM produces plausible-sounding Arabic that actually means something different | Cosine similarity via MiniLM/LaBSE; Green=approve, Yellow=escalate, Red=redo | MiniLM (ONNX), LaBSE, IVectorEncoder |
| **G** | [Human Review (HITL)](./phase-g-hitl.md) | Routes uncertain translations to a human reviewer via a translation tool | Some idioms and technical nuances can't be verified by AI alone | Yellow/Red units pushed to Weblate/MateCAT/doccano; human corrects → webhook syncs back to DB | Weblate, MateCAT, doccano |
| **H** | [Restore & Publish](./phase-h-restore-publish.md) | Injects verified Arabic back into a fresh copy of the English HTML | The final step that produces the actual Arabic web page | PathID-based DOM traversal; unmask tokens; set `dir="rtl"`; optional hover-tooltip annotation | AngleSharp |

---

## External Resources

The pipeline relies on open-source datasets, NLP tools, and translation standards. These are not built — they are integrated.

→ **[View Full Resources Reference](./resources.md)**

| Category | Key Resources |
|----------|--------------|
| Knowledge Graphs | Wikidata (CC0), ConceptNet, DBpedia, BabelNet |
| Translation Memory | OPUS UN Corpus, MyMemory, Tatoeba |
| Arabic NLP | CAMeL Tools (MIT), Farasa, Arabic WordNet |
| Semantic Models | MiniLM, LaBSE, mUSE, XLM-RoBERTa, LASER |
| Standards | TBX (terminology), TMX (translation memory), Unicode CLDR |
| Review Tools | Weblate, MateCAT, doccano |

---

## Document Map

```
00-index.md                  ← You are here
├── phase-a-extract-map.md
├── phase-b-protect-mask.md
├── phase-c-glossary-mining.md
├── phase-d-translate.md
├── phase-e-validate.md
├── phase-f-semantic-gate.md
├── phase-g-hitl.md
├── phase-h-restore-publish.md
└── resources.md
```

---

## MVP vs Full Pipeline

| Feature | MVP (Sprint 1–2) | Full Pipeline |
|---------|-----------------|--------------|
| Extract + PathID | ✅ | ✅ |
| Masking + audit | ✅ | ✅ |
| LLM translation with context + glossary | ✅ | ✅ |
| Basic validation + restore + RTL | ✅ | ✅ |
| Semantic verification (traffic light) | — | ✅ |
| Duplicate detection + TM reuse | — | ✅ |
| HITL integration | — | ✅ |
| Full glossary mining automation | — | ✅ |

---

*This pipeline is a production-ready blueprint. It handles the messiness of HTML, the fragility of CLI commands, and the nuance of Arabic linguistics in a scalable, verifiable, and stateless way.*
