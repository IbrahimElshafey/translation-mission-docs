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
