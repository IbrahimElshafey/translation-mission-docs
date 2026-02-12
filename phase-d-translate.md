# Phase D — Translate

*Pipeline position: Fourth · Depends on: Phase B (masked text) + Phase C (approved glossary) · Feeds into: Phase E (Validate)*

---

## Objective

Send each masked translation unit to an LLM with a carefully structured prompt that includes context, glossary, and style rules. Store the Arabic Markdown result. The orchestrator manages chunking, rolling context, caching, and retries.

---

## Inputs & Outputs

| | Value |
|--|-------|
| **Input** | `MaskedEnglishMD` + `Context` breadcrumb + approved `GlossaryEntry` table |
| **Output** | `ArabicMD` (masked Arabic Markdown) stored per `TranslationUnit` |
| **Primary tool** | LLM (e.g., Gemini 1.5 Pro / Flash) |
| **Orchestrator** | C# Translation Orchestrator service |

---

## Core Insight (2026 Reality)

Modern LLMs already solve basic translation quality. The real challenges are:

- **Terminology consistency** — without an explicit glossary, the LLM invents its own
- **Context continuity** — isolated paragraphs lose pronoun references and tone
- **Technical content protection** — the masking from Phase B handles this
- **Validation** — Phases E and F handle this

The Translation Orchestrator's job is **context and control**, not rewriting text.

---

## What Every LLM Request Must Contain

### 1. Current Unit (Masked Markdown)
The actual text to translate, with `[[M_x]]` placeholders already in place:
```
Run [[M_0]] to install the server, then edit [[M_1]] to set the [[M_2]] value.
```

### 2. Section Title + Previous Paragraph (Rolling Context)
Never translate in isolation. Include:
- The current section's heading
- The immediately preceding paragraph (already translated if available, or English if first)

This preserves pronoun references ("it", "this", "the above") and prevents tone drift across paragraphs.

### 3. Context Breadcrumb (Tone Signal)
The `Context` field from Phase A:
```
section.docs-content > div.warning > p
```
This tells the LLM: "This is a warning box inside technical documentation — use formal, precise Arabic. No creative phrasing."

### 4. Approved Glossary (Subset)
Include only the **relevant terms** for this unit — not the entire glossary. Filter by terms that appear in the unit text:

```json
{
  "approved_translations": {
    "dependency injection": "حقن الاعتماديات",
    "middleware": "البرمجيات الوسيطة",
    "DbContext": "KEEP_ENGLISH"
  },
  "keep_english": ["AngleSharp", "FastText", "SignalR", ".NET"]
}
```

### 5. Style Rules (Consistent Across All Requests)
Define once, inject in every prompt:

```
- Use formal Modern Standard Arabic (MSA)
- Active voice preferred; passive only for imperative instructions
- Technical terms in the approved_translations list must be used exactly as shown
- Terms in keep_english list must appear in Arabic text unchanged
- Do not translate Markdown syntax: **bold**, [link](url), `code`
- Preserve all [[M_x]] placeholders exactly as-is — do not modify, move, or remove them
- Sentence length: match the English length; do not expand with explanatory phrases
- "must" → يجب, "should" → ينبغي, "may" → يجوز — keep modal consistency
```

---

## Prompt Structure (Template)

```
SYSTEM:
You are a professional technical Arabic translator. Translate the user's content
from English to Arabic following the style rules and glossary provided.

STYLE RULES:
{style_rules}

GLOSSARY (use exactly):
{filtered_glossary_json}

CONTEXT:
- Section: {section_title}
- Element type: {context_breadcrumb}
- Previous paragraph (for reference, do not translate): {previous_paragraph}

TASK:
Translate the following Markdown text to Arabic.
Rules:
1. Keep all [[M_x]] placeholders exactly unchanged.
2. Keep all Markdown syntax (**, [], (), ``) unchanged.
3. Apply glossary terms exactly as specified.
4. Return ONLY the Arabic Markdown. No explanations.

TEXT TO TRANSLATE:
{masked_english_md}
```

---

## Orchestrator Responsibilities

### Chunking
For long sections (e.g., a page with 50 paragraphs), process in order from top to bottom. The rolling context window automatically builds as translation progresses.

### Rolling Context Window
Maintain a buffer of the last 2 translated paragraphs. Include the most recent in every new request. This is the **single biggest quality improvement** over batch translation.

### Translation Memory Cache
Before sending to the LLM, check the Translation Memory (from Phase C duplicate detection):
- Exact hash match → skip LLM, reuse stored translation → massive cost saving
- No match → proceed with LLM call

### Retry Policy

| Scenario | Action |
|----------|--------|
| Phase E audit fails (placeholder corruption) | Retry with stricter explicit placeholder rules |
| Phase F returns Red | Retry with `temperature=0` and literal translation constraint |
| Phase F returns Yellow | Optional: retry with rephrased prompt; or escalate to Phase G |
| LLM API error / timeout | Exponential backoff, max 3 retries |

### Cost Management
- Use a fast/cheap LLM model for standard content (e.g., Gemini Flash)
- Reserve the high-reasoning model (e.g., Gemini Pro) for Yellow/Red retries only
- Translation Memory hits cost zero — maximize duplicate detection

---

## Database Updates

```sql
UPDATE TranslationUnit SET
    ArabicMD           = '...',            -- LLM output
    ModelVersion       = 'gemini-flash-2', -- Track which model was used
    GlossaryVersion    = 'v1.2.0',         -- Track which glossary was used
    TranslationStatus  = 'Translated'
WHERE PathID = '1,0,2';
```

---

## Self-Audit Pass (Optional but Recommended)

After the main translation, run a **fast secondary LLM pass** (cheaper model):

```
TASK: Review this translation pair.
English: {original_english}
Arabic: {arabic_output}

Check ONLY:
1. Are all [[M_x]] placeholders present and unmodified? (yes/no)
2. Are there obvious grammar errors? (yes/no)
3. Is any English word translated that should be kept in English per the keep_english list? (yes/no)

Reply with: PASS or FAIL:[reason]
```

A `FAIL` result triggers the retry policy before Phase E even runs.

---

## Success Criteria

- Every `TranslationUnit` with status `MaskingComplete` has a non-empty `ArabicMD`
- All placeholder parity pre-checks pass before reaching Phase E
- Rolling context is maintained throughout a page's translation
- Glossary terms are applied consistently (verified in Phase E)
- Translation Memory reduces LLM calls by at least 10–20% on most sites

---

## 💡 Architect Suggestions

**1. Never translate paragraphs in isolation**
Providing the previous paragraph is the single most impactful change over naive translation. It resolves "this", "it", "the above approach" references correctly and maintains tone continuity. If you can only implement one context feature, make it this one.

**2. Implement a rolling glossary injection filter**
Don't include all 500 glossary terms in every prompt — LLMs have context limits and long glossaries degrade instruction-following. Extract only the subset of terms that appear in the current unit. A simple `Contains()` check before building the prompt is sufficient.

**3. Store the exact prompt sent per unit (for debugging)**
During development and QA, store the full prompt that was sent in a `DebugPrompt` field. When a translation fails Phase F, you can immediately see what instructions the LLM received. This dramatically speeds up debugging.

**4. Use separate temperature settings per content type**
- UI strings, navigation labels: `temperature=0` (deterministic, consistent)
- Technical documentation: `temperature=0.2` (mostly deterministic, slight flexibility)
- Marketing/hero text: `temperature=0.4` (allow natural-sounding Arabic)
Tag content type in the `Context` field during Phase A and use it here.

**5. Batch API calls where supported**
Most LLM providers support batching multiple requests. Group 10–20 short units per API call to reduce network latency and lower costs. The orchestrator should have a configurable `BatchSize` parameter.

**6. Track glossary compliance rate as a metric**
After Phase E, calculate: (glossary terms correctly applied / total glossary term occurrences). Track this per model version and per glossary version. A compliance rate below 90% indicates the prompt structure needs adjustment or the model needs to be upgraded.
