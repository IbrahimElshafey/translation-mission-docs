# Phase C — Glossary Mining & Term Intelligence

*Pipeline position: Third (runs before any translation) · Depends on: All extracted TranslationUnits · Feeds into: Phase D (Translate)*

---

## Objective

Before a single sentence is sent to the LLM, analyse the entire site to discover, rank, and lock down technical terminology. The approved glossary becomes a contract: every approved term is translated exactly the same way across all pages.

---

## Inputs & Outputs

| | Value |
|--|-------|
| **Input** | All `TranslationUnit.EnglishMD` records + site link structure |
| **Output** | `GlossaryEntry` table: term → approved Arabic translation or "keep English" |
| **Primary tools** | spaCy, GLiNER, gensim, NLTK, TF-IDF |

---

## Why Glossary Mining Must Come First

Without a pre-built glossary:
- "dependency injection" might be translated four different ways across four pages
- A term like "middleware" might be translated on page 1 but transliterated on page 2
- The LLM has no anchor for your specific domain vocabulary

Mining the glossary first means the LLM receives **explicit instructions** rather than making its own terminology decisions.

---

## The 80/20 Principle

> 20% of the terms in your documentation account for 80% of the translation quality problems.

These techniques automatically find that critical 20%. Human effort is then focused only on reviewing the top 200–500 candidates — not reading every page.

---

## Mining Techniques

### Technique 1 — Hash-Based Duplicate Detection

Compute a normalized hash of each `EnglishMD` block. Identical hashes across pages = the same content block.

**Result:** Translate once, reuse everywhere. Never send the same navigation menu or disclaimer to the LLM 47 times.

```
Block: "This feature is in preview and subject to change."
Hash:  a7f3c9e2...
Pages: 47 pages contain this exact block
Action: Translate once → store in Translation Memory → inject on all 47 pages
```

---

### Technique 2 — TF-IDF (Term Frequency × Inverse Document Frequency)

Finds terms that are **important to a specific page** but rare across the whole site — these are your domain-specific technical terms.

```
Page: "Entity Framework Core Guide"
High TF-IDF terms: DbContext, migration, LINQ, EF Core
Low TF-IDF terms: database, query, entity  ← too common, not distinctive
```

**Glossary extraction:** Collect top 10–20 TF-IDF terms per page. Terms appearing in the top 10 of multiple pages = core technical vocabulary that needs the glossary.

---

### Technique 3 — Document Clustering

Group semantically similar pages using K-means on TF-IDF vectors. Each cluster shares vocabulary that must be consistent.

```
Cluster: "Authentication & Security" (8 pages)
Shared terms: JWT, OAuth, bearer token, claim, scope
→ All 8 pages must use the same Arabic translation for these terms

Cluster: "Database Operations" (12 pages)
Shared terms: DbContext, migration, LINQ, connection string
→ Separate set of consistent terms
```

---

### Technique 4 — Named Entity Recognition (NER)

Identifies technologies, products, companies, and frameworks that should **stay in English** or follow a specific transliteration rule.

**Recommended hybrid approach (no GPU needed):**

1. **Dictionary/rule-based first** (FlashText + EntityRuler): catches 80% of technical terms instantly
2. **Small spaCy model second** (`en_core_web_sm`): catches remaining general proper nouns
3. **GLiNER optional third** (see Suggestions): transformer-based, for gaps the above miss

**NER entity decisions:**
| Entity Type | Example | Decision |
|-------------|---------|---------|
| Technology / Framework | `.NET`, `React`, `Kubernetes` | Keep in English |
| Product | `Azure App Service`, `GitHub` | Keep in English |
| Company | `Microsoft`, `Google` | Keep in English |
| Technical class/method | `DbContext`, `IVectorEncoder` | Keep in English or transliterate |
| Conceptual term | "dependency injection", "middleware" | Translate with approved Arabic |

---

### Technique 5 — Co-Occurrence Analysis (Collocation Detection)

Finds multi-word technical terms that must be treated as a single unit — not translated word by word.

**Without collocation detection:**
- "dependency" → `تبعية`
- "injection" → `حقن`
- Combined → `تبعية حقن` (meaningless in Arabic technical context)

**With collocation detection:**
- "dependency injection" → single glossary entry → `حقن الاعتماديات` or kept as-is

**Statistical measures used:**
- PMI (Pointwise Mutual Information): how much more likely are these words together than apart?
- Log-likelihood ratio: significance of the co-occurrence
- Chi-square test: independence test for word pairs

**Tools:** gensim `Phrases`, NLTK collocations, spaCy n-gram analysis

---

### Technique 6 — Link Graph Analysis (PageRank-Style)

Pages with many incoming internal links contain the most important terminology. Rank them by in-degree centrality:

```
/concepts/dependency-injection.html  ← linked from 23 pages  → Priority: CRITICAL
/concepts/middleware.html            ← linked from 18 pages  → Priority: HIGH
/tutorials/hello-world.html          ← linked from 2 pages   → Priority: STANDARD
```

**Result:** A ranked priority list that tells human reviewers where to focus their glossary review effort first.

---

### Technique 7 — Cross-Reference Detection

Find terms that have a canonical definition page somewhere in the site and are referenced from many other pages. These are always high-priority glossary entries because their meaning is authoritative and central.

```
Definition: /concepts/middleware.html → "software assembled into an app pipeline..."
References: 25 uses across 3 other pages
→ Glossary priority: HIGH; translation must match the canonical definition
```

---

## Phase Output: Glossary Table

After running all techniques, merge results and score each term:

| Field | Description |
|-------|-------------|
| `Term` | English term (single or multi-word) |
| `Frequency` | Total occurrences across site |
| `TFIDFScore` | Average TF-IDF importance score |
| `PageRankScore` | Importance of pages where it appears |
| `Category` | Technology / Concept / UI / Legal |
| `ProposedArabic` | Auto-suggested translation (from ConceptNet/Wikidata lookup) |
| `Decision` | `TranslateArabic` / `KeepEnglish` / `Transliterate` |
| `ApprovedBy` | Human reviewer name |
| `GlossaryVersion` | Version hash — linked to translation runs |

**Human review step:** Present the top 200–500 highest-scored terms to a human reviewer. The reviewer assigns a decision for each. This is the single most impactful step for translation quality.

---

## Computing Resources

| Technique | CPU Need | GPU Need | Time (1000 pages) |
|-----------|---------|---------|-------------------|
| Hash deduplication | Minimal | None | Seconds |
| TF-IDF | Low | None | 1–3 min |
| Clustering (K-means) | Moderate | None | 2–5 min |
| NER (dictionary + spaCy) | Low | None | 5–15 min |
| Co-occurrence (gensim) | Moderate | None | 3–8 min |
| Link graph (NetworkX) | Low | None | 1–2 min |

**Total pre-translation analysis on a standard 8-core CPU with 8GB RAM: ~20–40 minutes for a 1000-page site.**

---

## Success Criteria

- Top 200–500 terms have been reviewed and approved by a human
- All approved terms are stored in the `GlossaryEntry` table with a `GlossaryVersion`
- Duplicate content blocks are detected and their single translation is stored in TM
- Every Phase D LLM prompt includes the relevant subset of the approved glossary

---

## 💡 Architect Suggestions

**1. Use GLiNER for NER instead of (or after) spaCy**
[GLiNER](https://github.com/urchade/GLiNER) is a state-of-the-art zero-shot NER model that can detect arbitrary entity types without fine-tuning. Unlike spaCy which only detects fixed entity types (PERSON, ORG, GPE…), GLiNER lets you define custom categories like `"programming framework"`, `"CLI command"`, `"file path"`. This is especially useful for technical documentation where the entity types don't match standard NLP training sets. It runs on CPU and handles inference in reasonable time for one-time pre-translation analysis.

**2. Filter stop words aggressively before co-occurrence analysis**
Raw vocabulary for a 1000-page site might be 50,000 unique tokens. After removing stop words, punctuation, and terms with fewer than 3 occurrences, you're down to 5,000–10,000 meaningful terms. Co-occurrence analysis on the filtered set takes minutes, not hours. Use `spacy.lang.en.stop_words` + a custom technical stop list (words like "function", "method", "example" that are common but not glossary-worthy).

**3. Use gensim `Phrases` with two passes for bigrams and trigrams**
First pass: detect bigrams (2-word collocations). Second pass: run `Phrases` again on the bigram-transformed corpus to detect trigrams. This correctly identifies `"dependency injection container"` as a 3-word unit rather than just `"dependency injection"` + `"container"`.

**4. Version your glossary and link it to translation runs**
Store a `GlossaryVersion` (hash of the glossary table contents) in each `TranslationUnit` record at translation time. If the glossary is updated later, you can identify which records need re-translation without re-processing the whole site.

**5. Cross-check proposed Arabic translations against Wikidata and ConceptNet**
Before presenting terms to the human reviewer, auto-populate `ProposedArabic` by querying Wikidata SPARQL for the Arabic label of matching entities and ConceptNet for Arabic equivalents. This saves reviewers significant time — they validate rather than invent translations.

**6. Build a "keep English" fast list from your brand dictionary**
Any term already in your Phase B brand/proper-noun masking dictionary should automatically be assigned `KeepEnglish` in the glossary, bypassing human review. This eliminates an entire category of review work.
