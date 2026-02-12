# Phase E — Post-Translation Validation

*Pipeline position: Fifth · Depends on: Phase D (ArabicMD) · Feeds into: Phase F (Semantic Gate)*

---

## Objective

Run automated structural and correctness checks on the Arabic output immediately after translation. Catch problems that are deterministically verifiable before spending compute on semantic analysis.

---

## Inputs & Outputs

| | Value |
|--|-------|
| **Input** | `ArabicMD` + `MaskedEnglishMD` + `ProtectionManifest` + `GlossaryEntry` table |
| **Output** | `ValidationStatus` per record: `Passed` / `HardFail` / `SoftFail` |

---

## Hard Failures (Reject Immediately, Retry Phase D)

| Check | What It Verifies | Detection Method |
|-------|-----------------|-----------------|
| **Placeholder parity** | `[[M_x]]` count in Arabic == count in English | Token count comparison |
| **Placeholder corruption** | No `[[M_x]]` was altered (`[[ M_0 ]]`, `[[م_0]]`, `[M_0]`) | Strict regex match |
| **Placeholder leak** | No original protected string appears outside its token | Scan Arabic for manifest values |
| **Missing content** | Arabic output is not empty / not shorter than 10% of English | Length ratio check |
| **Broken links** | Markdown `[text](url)` links are structurally intact | Markdown parser check |
| **Broken list structure** | List items count matches between English and Arabic | List item count comparison |
| **Protected token translated** | A `KeepEnglish` glossary term appears translated | Glossary enforcement check |

A single hard failure → `ValidationStatus = 'HardFail'` → send back to Phase D orchestrator for retry.

---

## Soft Failures (Escalate, Do Not Retry Automatically)

| Check | What It Verifies |
|-------|-----------------|
| **Glossary term missing** | An approved translation should have been used but wasn't |
| **Unusual structure change** | Arabic paragraph count differs from English |
| **Excessive length change** | Arabic is >200% longer or <40% of English character count |

Soft failures → `ValidationStatus = 'SoftFail'` → flag for Phase F to weigh against semantic score.

---

## Optional: Fast LLM Self-Audit

After structural checks pass, an optional secondary LLM call (cheap model) checks:
- Grammar validity
- Placeholder safety confirmation
- Obvious non-sequiturs

This runs only on units that passed all structural checks. A `FAIL` from the self-audit downgrades to `SoftFail`.

---

## 💡 Architect Suggestions

**1. Run Phase E synchronously before writing ArabicMD to the main DB**
Never commit a translation to the main table if it fails a hard check. Write to a staging table first, run Phase E, then promote to main only on `Passed`. This keeps the main DB clean and avoids partial data issues.

**2. Build a "failure sample library" for unit tests**
Create a set of known-bad LLM outputs (corrupted placeholders, missing list items, leaked technical terms) and use them as unit tests for the validation engine. The original spec recommended this — it is essential for regression testing when the validation rules are updated.

**3. Log every failure with the specific failing check name**
Don't just log `ValidationStatus = 'HardFail'`. Log `FailedCheck = 'PlaceholderParity'`, `Expected = 3`, `Actual = 2`. This data drives prompt engineering improvements — if 30% of failures are parity failures, add a stronger placeholder instruction to the Phase D prompt.

---

---

# Phase F — Semantic Verification Gate

*Pipeline position: Sixth · Depends on: Phase E (Passed units) · Feeds into: Phase G (HITL) or Phase H (Restore)*

---

## Objective

Mathematically verify that the Arabic translation preserves the **meaning** of the English source. This catches hallucinations, context drift, and creative rewriting that look structurally fine but say something different.

---

## Inputs & Outputs

| | Value |
|--|-------|
| **Input** | `EnglishMD` (unmasked) + `ArabicMD` (unmasked) |
| **Output** | `SemanticScore` (float), `VerificationStatus`, `IsVerified` (bool) |
| **Primary tool** | MiniLM L12 via ONNX Runtime (local, no API cost) |

---

## The Core Engine

**Model:** `paraphrase-multilingual-MiniLM-L12-v2`

Maps 50+ languages (including English and Arabic) into a **shared 384-dimensional vector space**. If the English and Arabic sentences mean the same thing, their vectors will be numerically close.

**Deployment:** ONNX Runtime via `Microsoft.ML.OnnxRuntime` in C#. No Python dependency. Runs entirely on-premises. Target speed: **<15ms per unit** on a standard server CPU.

---

## Step-by-Step Process

### Step 1 — Clean Input
Before encoding, strip `[[M_x]]` placeholders from both strings. Placeholder tokens create "vector noise" that skews the similarity score.

```csharp
var cleanEnglish = StripPlaceholders(unit.EnglishMD);
var cleanArabic  = StripPlaceholders(unit.ArabicMD);
```

### Step 2 — Generate Vectors
```csharp
float[] vectorEN = encoder.Encode(cleanEnglish);
float[] vectorAR = encoder.Encode(cleanArabic);
```

### Step 3 — Calculate Cosine Similarity
```
CosineSimilarity(A, B) = (A · B) / (||A|| × ||B||)
Result range: 0.0 (completely different) → 1.0 (identical meaning)
```

---

## Traffic Light Decision

| Score (θ) | Status | Action |
|-----------|--------|--------|
| **θ ≥ 0.85** | 🟢 **Green** | Auto-approve. Set `IsVerified = true`. |
| **0.70 ≤ θ < 0.85** | 🟡 **Yellow** | Escalate. Send to LLM Idiom Audit or Phase G HITL. |
| **θ < 0.70** | 🔴 **Red** | Reject. Re-translate with `temperature=0` and literal constraints. |

### Yellow Zone — Idiom Audit
Some valid translations naturally score lower because they use different linguistic roots (e.g., English idiom "bare minimum" → Arabic "الحد الأدنى" uses structurally different words). Before escalating to human review, send a reasoning prompt to a high-capability LLM:

```
English: [text]
Arabic: [text]
Similarity score: 0.76

Is this a valid technical translation using Arabic idioms, or is it a meaning failure?
Reply VALID or REDO with one-line reason.
```

- `VALID` → promote to Green, log `VerificationLog = 'Idiomatic Approval'`
- `REDO` → demote to Red, retry Phase D

---

## Database Updates

```sql
ALTER TABLE TranslationUnit ADD COLUMN (
    SemanticScore    FLOAT,     -- Raw cosine similarity
    VerificationLog  TEXT,      -- e.g. 'Idiomatic Approval', 'Hallucination detected'
    IsVerified       BOOLEAN DEFAULT 0
);
```

---

## Model Options (Swappable via Strategy Pattern)

The verification layer is implemented as `IVectorEncoder` — model is a configuration choice, not a code change.

| Model | Size | Dims | Best For | C# Difficulty |
|-------|------|------|----------|---------------|
| **MiniLM L12** | ~100MB | 384 | Speed & efficiency (default) | Very Easy (ONNX) |
| **LaBSE** | ~500MB | 768 | Bitext alignment — high-priority pages | Moderate (ONNX) |
| **mUSE** | ~1GB | 512 | Idiomatic / conceptual similarity | Hard (TensorFlow) |
| **XLM-RoBERTa** | ~1.1GB | 768 | Maximum accuracy | Moderate (ONNX) |
| **LASER** | ~300MB | 1024 | Tight cross-lingual clusters | Moderate (ONNX) |

**Recommendation:** Use MiniLM as default. Switch to LaBSE for legal pages, landing pages, or any content where a meaning error would be costly.

### Strategy Pattern Interface
```csharp
public interface IVectorEncoder
{
    float[] Encode(string text);
    int Dimensions { get; }  // 384 for MiniLM, 768 for LaBSE
}
```
Model path and dimension are injected via configuration — no code change required to swap models.

---

## Success Criteria

1. A 1:1 literal translation scores ≥ 0.90
2. A hallucinated paragraph (text added that wasn't in the English) scores ≤ 0.65 (caught as Red)
3. Single-unit verification runs in <15ms on standard CPU
4. Yellow zone idiom audit correctly approves valid idiomatic translations

---

## 💡 Architect Suggestions

**1. Calibrate thresholds per content type, not globally**
Legal/compliance text: raise the Green threshold to 0.90. Marketing text: lower to 0.80 (more creative phrasing is acceptable). UI button labels: skip semantic verification entirely (too short for meaningful cosine similarity). Store `ContentType` in the `TranslationUnit` and apply type-specific thresholds.

**2. Run embeddings in batches for throughput**
ONNX Runtime supports batch inference. Process 32–64 units per batch to maximize CPU/GPU throughput. Cold start (model loading) is the expensive part — amortize it across a full batch job.

**3. Use LaBSE for "bitext mining" quality assurance**
After a full site translation run, run a batch verification pass using LaBSE (higher accuracy than MiniLM) for all Green units that scored between 0.85–0.90. This secondary pass catches "borderline Green" units that MiniLM approved but LaBSE would flag.

**4. Store the model version used for each verification**
When you upgrade from MiniLM to LaBSE, units verified with the old model should be re-verified. Store `VerificationModelVersion` so you can query which units need re-checking after a model upgrade.

**5. The Yellow zone is your most valuable signal**
Yellow units are not failures — they are the most interesting cases. Track them separately. Over time, patterns in Yellow units reveal where your prompt engineering or glossary needs improvement (e.g., a specific domain cluster consistently scores Yellow).

**6. XLM-RoBERTa for detecting subtle technical nuance**
If MiniLM consistently fails to distinguish between closely related terms (e.g., "Service Discovery" vs "Service Registry"), use XLM-RoBERTa for a second-pass verification on technical documentation clusters. It is expensive for a full site but can be targeted at specific page clusters.
