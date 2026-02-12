
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
