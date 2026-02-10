This document specifies the implementation of the **Semantic Verification Layer**. Its purpose is to ensure that the Arabic translation maintains the "mathematical meaning" of the English source before it is committed to the production database.

---

## Developer Task: Cross-Lingual Semantic Verification

### 1. Objective

To implement an on-premises **Quality Gate** using the `paraphrase-multilingual-MiniLM-L12-v2` model. This module must calculate the semantic distance between the English source and the Arabic translation to detect "Meaning Drift," hallucinations, or structural failures.

### 2. The Core Engine: Multilingual MiniLM

You will use a transformer-based embedding model that maps 50+ languages (including Arabic and English) into a **shared 384-dimensional vector space**.

* **Property:** If English Sentence  and Arabic Sentence  have the same meaning, their vectors will be numerically close (High Cosine Similarity).
* **Model Deployment:** For C# compatibility, the model should be executed via **ONNX Runtime** to ensure high-performance, local inference without a Python dependency.

---

### 3. Phase I: Vector Encoding (Pre-Storage)

For every `TranslationUnit` processed by the LLM:

1. **Clean Input:** Strip the `[[M_x]]` placeholders from both the English and Arabic strings to prevent "token noise" from skewing the vector.
2. **Generate Vectors:**
*  = `Model.Encode(Source_English_Text)`
*  = `Model.Encode(Target_Arabic_Text)`


3. **Calculate Similarity:**
Apply the Cosine Similarity formula:



---

### 4. Phase II: Logic & Action Thresholds

The verification system must act as a "Traffic Light" based on the similarity score ():

| Score () | Status | Action |
| --- | --- | --- |
| **** | **Green** | Auto-approve. High confidence that meaning is preserved. |
| **** | **Yellow** | **Escalate.** Trigger an LLM "Self-Audit" pass to confirm if the difference is due to an idiom/creative phrasing. |
| **** | **Red** | **Reject.** Potential hallucination or total context loss. Request a re-translation with higher temperature or literal constraints. |

---

### 5. Phase III: Post-Pipeline Verification Storage

The developer must update the database to store the "Semantic Health" of each record.

**Database Schema Update:**

```sql
ALTER TABLE TranslationUnit ADD COLUMN (
    SemanticScore FLOAT,        -- The raw Cosine Similarity
    VerificationLog TEXT,       -- Explanation for "Yellow" pass (e.g., 'Idiomatic Approval')
    IsVerified BOOLEAN DEFAULT 0
);

```

---

### 6. Handling "Idiomatic Exceptions" (The Human-in-the-Loop)

If a block is flagged as **Yellow**, the system must not discard it. Some technical idioms (e.g., "bare minimum") will naturally have a lower score because their Arabic equivalent ("الحد الأدنى") uses different linguistic roots.

**The Auditor Logic:**

* If  is in the Yellow zone, send a prompt to a high-reasoning LLM (e.g., Gemini 1.5 Pro):
> "The similarity score for this translation is low. English: [EN] | Arabic: [AR]. Is this a valid technical translation using Arabic idioms, or is it a failure? Reply 'VALID' or 'REDO'."



---

### 7. Technical Constraints for the Developer

* **ONNX Implementation:** Use `Microsoft.ML.OnnxRuntime` and a C# BertTokenizer to process the text.
* **Cold Start:** Embeddings should be generated in batches to maximize CPU/GPU throughput.
* **Statelessness:** The verification must work even if the translation was performed 24 hours ago; only the EN and AR text strings are required for the check.

---

### 8. Success Criteria

1. **Hallucination Catching:** The system correctly flags a case where the LLM adds a paragraph that wasn't in the English source.
2. **Accuracy:** A 1:1 literal translation should yield a score of .
3. **Speed:** Verification of a single block must take less than 15ms on a standard server CPU.

---

**This completes the technical documentation. Would you like me to now synthesize all of this into a high-level "System Architecture Overview" for your Lead Architect?**
