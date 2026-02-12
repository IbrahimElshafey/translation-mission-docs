# External Resources Reference

*All open-source tools, datasets, models, and standards used in the pipeline*

---

## How to Use This Page

Each resource is listed with:
- **What it is** — a plain-English description
- **How/where to use it** — which pipeline phase uses it and what for
- **Access** — URL, API, or download
- **License** — important for commercial deployment

---

## Semantic Embedding Models (Phase F)

| Name | What It Is | How/Where to Use | Size | License |
|------|-----------|-----------------|------|---------|
| **MiniLM L12** (`paraphrase-multilingual-MiniLM-L12-v2`) | Fast multilingual sentence encoder; 50+ languages in shared 384-dim vector space | Phase F — default semantic verifier; run via ONNX Runtime in C# | ~100MB | Apache 2.0 |
| **LaBSE** (Language-Agnostic BERT Sentence Embedding) | Google model trained for cross-lingual bitext alignment across 109 languages | Phase F — use for high-priority pages (legal, landing pages) where meaning drift is most costly | ~500MB | Apache 2.0 |
| **mUSE** (Multilingual Universal Sentence Encoder) | Google model; less sensitive to word-for-word matching; better at "big picture" intent | Phase F — Yellow zone idiom cases where structural differences are expected | ~1GB | Apache 2.0 |
| **XLM-RoBERTa** | Heavy transformer; state-of-the-art for Arabic NLP tasks | Phase F — second-pass verification for technical nuance; too slow for every block | ~1.1GB | MIT |
| **LASER** (Meta, Bi-LSTM) | Single encoder for all languages; tight cross-lingual clusters; prefers sentences | Phase F — alternative to MiniLM for short sentence-level units | ~300MB | BSD |

**Interface:** All models are used via `IVectorEncoder` strategy pattern — swap by config, no code changes.

---

## NER & Text Mining Tools (Phase C)

| Name | What It Is | How/Where to Use | Access | License |
|------|-----------|-----------------|--------|---------|
| **GLiNER** | Zero-shot NER model — detects custom entity types without fine-tuning (e.g., "programming framework", "CLI command") | Phase C — NER pass for technical terms that don't match standard NER categories | [github.com/urchade/GLiNER](https://github.com/urchade/GLiNER) | Apache 2.0 |
| **spaCy** (small model `en_core_web_sm`) | Industrial-strength NLP library with fast NER, tokenization, and dependency parsing | Phase C — second-pass NER for general proper nouns after dictionary pass | `pip install spacy` | MIT |
| **FlashText** | Keyword extraction and replacement at O(n) speed regardless of dictionary size | Phase B — brand/proper noun masking dictionary; Phase C — term frequency counting | [github.com/vi3k6i5/flashtext](https://github.com/vi3k6i5/flashtext) | MIT |
| **gensim `Phrases`** | Bigram/trigram collocation detection; fast and production-ready | Phase C — detect multi-word technical terms like "dependency injection container" | `pip install gensim` | LGPL |
| **NLTK collocations** | Classic NLP library for collocation and n-gram analysis | Phase C — PMI, log-likelihood co-occurrence scoring | `pip install nltk` | Apache 2.0 |
| **scikit-learn TF-IDF** | TF-IDF vectorizer; efficient for large corpora | Phase C — identify distinctive terms per page | `pip install scikit-learn` | BSD |
| **FastText** (`lid.176.bin`) | Language detection model — 176 languages, ~900KB, microsecond inference | Phase A — triage: is this block English, Arabic, or code? | [fasttext.cc](https://fasttext.cc) | MIT |

---

## Knowledge Graphs & Semantic Databases (Phase C — Glossary enrichment)

| Name | What It Is | How/Where to Use | Access | License |
|------|-----------|-----------------|--------|---------|
| **Wikidata** | Structured knowledge base; 300+ language labels for 100M+ entities | Phase C — auto-populate proposed Arabic translations for detected entities | SPARQL: `query.wikidata.org` · REST API | CC0 (Public Domain) |
| **ConceptNet** | Semantic network; word meanings, relationships, cross-lingual links | Phase C — find Arabic equivalents and semantic relationships for glossary terms | REST API: `api.conceptnet.io` · [github.com/commonsense/conceptnet5](https://github.com/commonsense/conceptnet5) | CC BY-SA 4.0 |
| **DBpedia** | Structured data extracted from Wikipedia; 125+ languages | Phase C — entity definitions and multilingual labels | SPARQL: `dbpedia.org/sparql` | CC BY-SA / GNU FDL |
| **BabelNet** | Merges Wikipedia, WordNet, Wiktionary into one multilingual lexicon; 500+ languages | Phase C — comprehensive multilingual word sense lookup | REST API (1000 queries/day free) · [babelnet.org](https://babelnet.org) | Non-commercial research (verify for commercial) |
| **Arabic WordNet (AWN)** | Arabic semantic network (~10,000 synsets); part of Open Multilingual WordNet | Phase C — Arabic synonym relationships for glossary enrichment | Via [globalwordnet](https://github.com/globalwordnet) | Open (verify per language) |

---

## Arabic NLP Tools (Phase C + Phase H)

| Name | What It Is | How/Where to Use | Access | License |
|------|-----------|-----------------|--------|---------|
| **CAMeL Tools** | Comprehensive Arabic NLP toolkit: morphological analysis, disambiguation, tokenization | Phase C — Arabic text normalization; Phase H — validate Arabic output morphology | [github.com/CAMeL-Lab/camel_tools](https://github.com/CAMeL-Lab/camel_tools) · `pip install camel-tools` | MIT ✅ Commercial OK |
| **Farasa** | Arabic NLP: segmentation, POS tagging, diacritization, NER | Phase C — Arabic morphology for glossary validation | [github.com/qcri/farasa](https://github.com/qcri/farasa) | Research use only ⚠️ |
| **AraBERT** | Pre-trained Arabic BERT models for embeddings and NER | Phase C/F — Arabic-specific embeddings when cross-lingual models underperform | HuggingFace hub · [github.com/aub-mind/arabert](https://github.com/aub-mind/arabert) | MIT ✅ |
| **Unicode CLDR** | Locale data for 200+ languages: date/time formats, number formats, UI strings | Phase H — standardized Arabic UI element translations, locale-specific formatting | [cldr.unicode.org](https://cldr.unicode.org) · [github.com/unicode-org/cldr](https://github.com/unicode-org/cldr) | Unicode License ✅ |

---

## Translation Memory & Parallel Corpora (Phase D — TM Cache)

| Name | What It Is | How/Where to Use | Access | License |
|------|-----------|-----------------|--------|---------|
| **OPUS UN Corpus** | Professionally translated UN documents; 6 languages including Arabic; very high quality | Phase D — seed Translation Memory with high-quality EN↔AR sentence pairs | [opus.nlpl.eu](https://opus.nlpl.eu) | Public |
| **MyMemory** | World's largest Translation Memory (30B+ words); EN↔AR included | Phase D — lookup existing translations before calling LLM | REST API (1000 words/day free) · [mymemory.translated.net](https://mymemory.translated.net) | Various |
| **Tatoeba** | 10M+ crowdsourced sentence translations; 400+ languages | Phase C/D — example phrase patterns for glossary validation | [tatoeba.org/en/downloads](https://tatoeba.org/en/downloads) | CC BY 2.0 FR |
| **TED Talks Corpus** (via OPUS) | TED subtitles in 100+ languages; modern conversational + technical | Phase D — contemporary phrasing patterns | Via [opus.nlpl.eu](https://opus.nlpl.eu) | CC |

---

## Human Review Tools (Phase G — HITL)

| Name | What It Is | How/Where to Use | Access | License |
|------|-----------|-----------------|--------|---------|
| **MateCAT** | Professional web-based CAT tool for post-editing MT output; built-in REST API | Phase G — send Yellow/Red units to human linguist via API job; receive webhook on completion | [matecat.com](https://matecat.com) · Docker self-hosted | Open Source |
| **Weblate** | Git-integrated TMS for continuous localization; strong review status workflow | Phase G — write translations to Git branch; human reviews "Needs Review" strings; auto-commit | [weblate.org](https://weblate.org) · Self-hosted or hosted | GPL / SaaS |
| **doccano** | Lightweight data labeling tool; Accept/Reject interface | Phase G — fast "pass/fail" grading for high-volume Yellow units | [github.com/doccano/doccano](https://github.com/doccano/doccano) · Docker | MIT |

---

## HTML Parsing & Markdown Tools (Phase A + Phase H)

| Name | What It Is | How/Where to Use | Access | License |
|------|-----------|-----------------|--------|---------|
| **AngleSharp** | C# HTML5 DOM parser; deterministic traversal and injection | Phase A — DOM crawl, PathID generation · Phase H — node targeting, Arabic HTML injection | [anglesharp.github.io](https://anglesharp.github.io) · NuGet | MIT |
| **ReverseMarkdown** | HTML → Markdown converter for .NET | Phase A — convert `InnerHtml` to Markdown before storing | NuGet | MIT |
| **Markdig** | Markdown → HTML converter for .NET; extensible | Phase H — convert `ArabicMD` back to HTML for injection | NuGet | BSD |
| **Microsoft.ML.OnnxRuntime** | ONNX model inference runtime for .NET/C# | Phase F — run MiniLM/LaBSE locally without Python dependency | NuGet | MIT |

---

## Terminology & Standards

| Name | What It Is | How/Where to Use | Access |
|------|-----------|-----------------|--------|
| **TBX** (TermBase eXchange) | XML standard for terminology/glossary exchange between CAT tools | Phase C — export approved glossary in TBX for import into Weblate/MateCAT/OmegaT | ISO standard; supported by all CAT tools |
| **TMX** (Translation Memory eXchange) | XML standard for TM exchange | Phase D — import/export translation memory between systems | Industry standard |
| **IATE** (EU Terminology DB) | 8M+ validated terms across all domains; 24 EU languages (no Arabic but useful for EN validation) | Phase C — validate English terminology decisions | [iate.europa.eu](https://iate.europa.eu) |

---

## License Compatibility Quick Reference

| Use Case | Safe Resources |
|----------|---------------|
| **Commercial deployment** | Wikidata (CC0), ConceptNet (CC BY-SA), CAMeL Tools (MIT), MiniLM (Apache), OPUS (mostly permissive), Unicode CLDR (Unicode License) |
| **Research/internal only** | BabelNet (non-commercial), Farasa (research), some LDC resources |
| **Always verify before deploy** | BabelNet, any LDC resource — licenses can change |

---

## Integration Priority

### Minimum viable set (covers 80% of needs):
1. **FastText** — language triage (Phase A)
2. **FlashText + spaCy** — NER and masking (Phase B, C)
3. **Wikidata + ConceptNet** — glossary auto-proposals (Phase C)
4. **OPUS UN Corpus** — Translation Memory seed (Phase D)
5. **MiniLM via ONNX** — semantic verification (Phase F)
6. **CAMeL Tools + Unicode CLDR** — Arabic output quality (Phase H)
7. **Weblate or doccano** — HITL review (Phase G)

### Phase 2 additions:
- GLiNER (better technical NER)
- LaBSE (higher-accuracy semantic verification for priority pages)
- MateCAT (professional linguist workflow)
- BabelNet (richer multilingual lexicon)

### Phase 3 (domain-specific):
- Domain glossaries (medical, legal, technical)
- Custom fine-tuned TM from accumulated HITL corrections
- XLM-RoBERTa for maximum semantic accuracy
