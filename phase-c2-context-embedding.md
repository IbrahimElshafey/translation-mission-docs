# Phase C2 — Context Creation & Embedding

*Pipeline position: Between Glossary Mining (C) and Translate (D)*
*Depends on: Phase A (TranslationUnits), Phase C (approved Glossary)*
*Feeds into: Phase D (Translate) and Phase F (Semantic Gate)*

---

## Objective

Before any translation begins, pre-compute and persist three things for every `TranslationUnit`:

1. A **structured context object** — the full prompt "payload" that Phase D will send to the LLM, assembled once and stored
2. A **semantic embedding vector** of the English source — used in Phase F for cosine comparison without re-encoding
3. A **Translation Memory (TM) similarity check** — look up existing translations that are semantically close, so Phase D can reuse rather than re-translate

This phase transforms raw extracted text into "translation-ready" units. Phase D becomes a thin execution layer — it just fires prompts and stores results.

---

## Why This Phase Exists

Without it, Phase D must do several expensive operations **per unit, per retry, per re-run:**

| Problem without this phase | Cost |
|---------------------------|------|
| Context assembly rebuilt from scratch on every translation call | CPU waste + inconsistency |
| English embeddings generated during Phase F (after translation) | Latency; no early TM matching |
| TM lookup happens at translation time, slowing the orchestrator | Throughput bottleneck |
| Glossary filtering done inline per-call | Repeated redundant computation |

Pre-computing here means Phase D is pure throughput: read context object → call LLM → store result.

---

## Inputs & Outputs

| | Value |
|--|-------|
| **Input** | All `TranslationUnit` records (with `MaskedEnglishMD`, `Context`, `PathID`) + approved `GlossaryEntry` table |
| **Output** | `ContextPacket` per unit (stored in DB) + `EmbeddingVector` per unit + TM match candidates |
| **Primary tools** | MiniLM ONNX, gensim, custom context builder |

---

## Output 1 — The Context Packet

A `ContextPacket` is the fully assembled, ready-to-send payload for Phase D. It is built **once** here and reused on every translation attempt (including retries).

### Fields

| Field | Source | Purpose |
|-------|--------|---------|
| `UnitText` | `MaskedEnglishMD` | The actual text to translate |
| `SectionTitle` | Nearest `<h1>`/`<h2>` ancestor (from Phase A DOM walk) | Rolling context — what section is this in? |
| `PrecedingUnit` | Previous `PathID` record (ordered by DOM position) | Rolling context — what came before? |
| `ContextBreadcrumb` | `Context` field from Phase A | Tone signal for LLM |
| `FilteredGlossary` | Terms from `GlossaryEntry` that appear in `UnitText` | Only relevant glossary terms — not the full list |
| `ContentType` | Derived from `ContextBreadcrumb` classification (see below) | Controls LLM temperature + threshold in Phase F |
| `StyleRuleSet` | Reference to the active style rule version | Which style guide applies |
| `TMCandidates` | Top 1–3 semantically similar existing translations (from TM lookup) | Gives LLM a reference example if available |

---

### Content Type Classification

Derive `ContentType` from the `ContextBreadcrumb` using a rule map. This single field controls downstream behaviour in both Phase D (temperature) and Phase F (similarity threshold):

| Context Pattern | ContentType | LLM Temp | Phase F Threshold |
|-----------------|-------------|----------|-------------------|
| `pre`, `code`, `kbd` | `Code` | Skip | Skip (never translate) |
| `section.warning`, `div.alert` | `Warning` | 0.0 | 0.92 |
| `.legal`, `.terms`, `.privacy` | `Legal` | 0.0 | 0.92 |
| `h1`, `h2`, `h3` (section headings) | `Heading` | 0.1 | 0.88 |
| `.hero`, `.tagline`, `.cta` | `Marketing` | 0.4 | 0.78 |
| `nav`, `.menu`, `.breadcrumb` | `UI` | 0.0 | 0.85 |
| `p`, `li`, `td` inside `.docs` | `Technical` | 0.2 | 0.85 |
| `p.bio`, `.about`, `.author` | `Biographical` | 0.3 | 0.80 |
| `alt`, `title`, `placeholder` | `Attribute` | 0.1 | 0.85 |
| *(fallback)* | `General` | 0.2 | 0.82 |

---

### Filtered Glossary Assembly

Do not send the entire glossary (500+ terms) to the LLM — it degrades instruction-following and wastes context tokens. Instead, filter at this stage:

```
FilteredGlossary = GlossaryEntry
    WHERE Term appears in UnitText (case-insensitive substring match)
    OR Term appears in SectionTitle
    LIMIT 25 terms
```

If more than 25 terms match, rank by `TFIDFScore DESC` and keep the top 25.

Store the filtered glossary JSON directly in the `ContextPacket`. Phase D just reads it — no filtering logic there.

---

### Preceding Unit Resolution (Rolling Context)

For rolling context, find the **immediately preceding sibling** in DOM order:

```sql
SELECT ArabicMD, EnglishMD
FROM TranslationUnit
WHERE PageUrl = @currentPageUrl
  AND PathID < @currentPathID   -- lexicographic DOM order
ORDER BY PathID DESC
LIMIT 1;
```

**Rules:**
- If the preceding unit has `ArabicMD` already (this is a re-run or page processed top-to-bottom), use the Arabic version — this is more coherent
- If not yet translated (first run, in order), use the English version as context
- If the preceding unit is a heading, promote it to `SectionTitle` instead of `PrecedingUnit`
- Cross-section boundary: if the preceding unit is in a different `<section>`, use the section heading only — don't bleed context across unrelated sections

---

## Output 2 — English Embedding Vector

Encode the **unmasked** `EnglishMD` using MiniLM (same model as Phase F) and store the vector:

```csharp
string cleanText   = StripPlaceholders(unit.EnglishMD);
float[] vector     = encoder.Encode(cleanText);
string vectorJson  = JsonSerializer.Serialize(vector);
```

**Why pre-compute here rather than in Phase F?**

- Phase F only has the Arabic text at that point — it must encode both sides anyway, but the English side never changes. Pre-computing here means Phase F only encodes Arabic, halving its embedding work.
- Enables TM similarity search (Output 3) without waiting for translation to complete.
- Enables batch encoding of the entire site in one efficient pass — better CPU/GPU utilisation than inline per-unit encoding during translation.

Store as `EnglishEmbeddingVector` (JSON float array or binary blob).

---

## Output 3 — Translation Memory Lookup

Use the pre-computed `EnglishEmbeddingVector` to query the Translation Memory for similar previously translated units.

### What Is the Translation Memory?

The TM is a table of already-approved EN→AR pairs — seeded from:
- Phase C hash-based exact duplicates (already translated once)
- OPUS UN Corpus / MyMemory imports
- Human-approved HITL corrections from Phase G (highest quality)
- All `IsVerified = true` units from previous translation runs

### Similarity Search

For each unit, find TM entries where cosine similarity to `EnglishEmbeddingVector` exceeds a threshold:

```sql
-- Conceptual query (use a vector DB or in-memory ANN index in practice)
SELECT tm.EnglishText, tm.ArabicText, tm.SimilarityScore, tm.Source
FROM TranslationMemory tm
WHERE CosineSimilarity(tm.EmbeddingVector, @currentVector) > 0.88
ORDER BY SimilarityScore DESC
LIMIT 3;
```

**Exact match (score = 1.0):** Skip Phase D entirely. Reuse the TM Arabic directly. Set `TranslationSource = 'TM-Exact'`.

**Near match (0.88–0.99):** Include in `TMCandidates` list in the `ContextPacket`. Phase D sends it to the LLM as a reference example: `"Here is a similar approved translation for reference — use its style and terminology."` This dramatically improves consistency for near-duplicate content.

**No match:** `TMCandidates` is empty. Phase D proceeds with a clean LLM call.

### Vector Index Strategy

For a 1000-page site with ~10,000 translation units, an in-memory ANN index is sufficient:

| Scale | Recommended Tool |
|-------|-----------------|
| < 50,000 units | In-memory cosine scan (simple loop, fast enough) |
| 50,000–500,000 | **FAISS** (Facebook ANN library) — CPU-only, C# via P/Invoke or Python sidecar |
| > 500,000 | **Qdrant** or **Weaviate** — standalone vector DB with REST API |

For most static site translation projects, the in-memory approach is entirely sufficient.

---

## Database Schema

```sql
ALTER TABLE TranslationUnit ADD COLUMN (
    ContextPacketJson       TEXT,    -- Full serialized ContextPacket
    ContentType             TEXT,    -- 'Technical', 'Legal', 'Marketing', etc.
    EnglishEmbeddingVector  TEXT,    -- JSON float[] from MiniLM
    TMMatchType             TEXT,    -- 'Exact', 'Near', 'None'
    TMCandidateJson         TEXT,    -- JSON array of top TM matches
    ContextStatus           TEXT     -- 'Ready', 'TMExact' (skip translation)
);
```

Units with `ContextStatus = 'TMExact'` are skipped by Phase D. They go directly to Phase H restoration.

---

## Processing Order

This phase must process units **in DOM order within each page** — top to bottom, section by section. The rolling context window (`PrecedingUnit`) depends on this order.

```sql
SELECT * FROM TranslationUnit
WHERE ContextStatus IS NULL
ORDER BY PageUrl, PathID;  -- PathID is a DOM coordinate — sorts correctly
```

Process page by page. Within each page, process top to bottom. This ensures `PrecedingUnit` is always resolved from an already-processed record.

---

## Batch Encoding Strategy

Run all English embeddings in a single ONNX batch before doing any ContextPacket assembly:

```
Pass 1 — Batch encode all EnglishMD (entire site) → store all vectors
Pass 2 — For each unit in DOM order: build ContextPacket + run TM lookup
```

This maximises ONNX throughput (batch size 32–64) and avoids interleaving embedding calls with DB reads.

**Estimated time for 10,000 units on a standard CPU:**
- Batch encoding (MiniLM): ~4–8 minutes
- TM lookup + ContextPacket assembly: ~2–5 minutes
- Total: ~10–15 minutes

---

## Success Criteria

- Every `TranslationUnit` with `MaskingComplete` status has a non-null `ContextPacket`
- All `EnglishEmbeddingVector` fields are populated
- Exact TM matches are correctly flagged as `TMExact` (Phase D skips them)
- `FilteredGlossary` in each `ContextPacket` contains only terms relevant to that unit
- `ContentType` is correctly classified for every unit (no nulls)
- Processing order preserves DOM sequence within each page

---

## 💡 Architect Suggestions

**1. Store `ContextPacketJson` as the single source of truth for what the LLM received**
During debugging, every Phase D failure can be immediately diagnosed by inspecting `ContextPacketJson`. You see exactly what glossary was injected, what context was provided, what TM candidate was suggested. This removes all ambiguity from "why did the LLM produce this output?" investigations.

**2. Use FAISS for TM similarity search even at small scale**
[FAISS](https://github.com/facebookresearch/faiss) (Facebook AI Similarity Search) indexes 384-dimensional MiniLM vectors and returns top-K results in microseconds. Even for 10,000 units, it is faster than a linear scan and scales gracefully as your TM grows with each successive translation run. It runs on CPU with no GPU needed.

**3. Seed the TM from OPUS UN Corpus before the first translation run**
Import a subset of the OPUS United Nations Corpus (English↔Arabic) into your TM table before running Phase C2. Encode all entries. This gives the TM coverage for common formal/technical phrases from day one — rather than starting empty. Even a few thousand high-quality sentence pairs dramatically improve near-match rate on first run.

**4. Add a `SectionBoundaryFlag` to prevent context bleed across topics**
When the `PrecedingUnit` is in a different section (different `<h2>` ancestor), flag `SectionBoundaryFlag = true` in the `ContextPacket` and use **only** the section heading as context — not the last paragraph from the previous section. This prevents the LLM from carrying over tone or pronoun references from an unrelated section.

**5. Treat `<h1>`/`<h2>` headings as context anchors, not just translation units**
When assembling `ContextPacket` for any unit, walk up the DOM ancestors to find the nearest heading and store it as `SectionTitle`. This heading propagates to all units within the section as their stable context anchor. A unit deep in a subsection always knows it is under "Authentication > Token Refresh" — not just floating in a document.

**6. Separate embedding model from verification model if needed**
Phase C2 uses MiniLM to build the TM index. Phase F uses MiniLM (or LaBSE) to verify translations. If you upgrade Phase F to LaBSE for higher accuracy, your TM index is still in MiniLM space — the two are **incompatible for cosine comparison**. Either: (a) keep both phases on the same model, or (b) store two vector columns (`MiniLMVector`, `LaBSEVector`) so you can independently upgrade each use case. Plan for this before the first full site run.

**7. Re-run this phase cheaply when the glossary is updated**
If the approved glossary changes (new terms added, decisions revised), only the `FilteredGlossary` portion of each `ContextPacket` needs rebuilding — not the embeddings. Make the ContextPacket assembly idempotent: `UPDATE ContextPacketJson SET FilteredGlossary = ...` without touching the embedding columns. Add a `GlossaryVersion` field to `ContextPacket` so you can query which units were built with an outdated glossary and need selective rebuild.
