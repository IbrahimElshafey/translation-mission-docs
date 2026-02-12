# Phase G — Human-in-the-Loop (HITL) Escalation

*Pipeline position: Seventh (conditional) · Triggered by: Yellow or Red from Phase F · Feeds into: Phase H (Restore)*

---

## Objective

Route translations that the system cannot confidently verify to a human reviewer via an open-source translation tool. Once the human approves or corrects the Arabic, the DB is updated and the unit proceeds to restoration.

---

## Inputs & Outputs

| | Value |
|--|-------|
| **Input** | Yellow/Red `TranslationUnit` records with `PathID`, `EnglishMD`, `ArabicMD`, `SemanticScore`, `Context` |
| **Output** | Corrected `ArabicMD` + `IsVerified = true` synced back to DB |
| **Review tools** | Weblate, MateCAT, or doccano |

---

## When HITL Is Triggered

| Trigger | Condition |
|---------|-----------|
| **Yellow unit** | `SemanticScore` between 0.70–0.85 AND LLM Idiom Audit returned `REDO` (or was skipped) |
| **Red unit** | `SemanticScore` < 0.70 AND retry in Phase D did not improve the score |
| **Repeated hard fail** | Phase D retry limit reached without passing Phase E |
| **Manual flag** | Any unit manually marked for review during QA |

---

## Review Tool Options

### A — MateCAT *(Best for professional linguists, side-by-side post-editing)*

A web-based CAT (Computer-Assisted Translation) tool designed specifically for post-editing machine-translated text.

**Workflow:**
1. C# service calls MateCAT REST API to create a "Job" with the translation pair
2. Human reviewer logs in, sees English source alongside Arabic draft
3. Reviewer edits the Arabic in-place and clicks "Complete"
4. C# service polls or receives webhook → syncs corrected Arabic to DB

**Best for:** Professional translators, high-volume review, side-by-side editing interface
**Hosting:** Self-hosted via Docker (free, open source)
**API:** REST (moderate integration effort)

---

### B — Weblate *(Best for developer-centric workflows)*

The standard tool for open-source project localization, with deep Git integration.

**Workflow:**
1. C# service writes Yellow/Red translations to a `.json` or `.po` file in a Git branch
2. Weblate detects new strings and marks them as "Needs Review"
3. Reviewer approves or edits in the Weblate UI
4. Weblate commits the corrected translation back to Git
5. C# service reads the updated file → syncs to DB

**Best for:** Dev teams already using Git, ongoing incremental updates, distributed review teams
**Hosting:** Self-hosted or Weblate.org hosted
**API:** Low effort (Git-based)

---

### C — doccano *(Best for quick Accept/Reject annotation)*

A lightweight data labeling tool. Not a full translation suite — just a fast "is this translation good?" interface.

**Workflow:**
1. Export Yellow/Red pairs as JSON
2. Human reviewer sees source + draft and clicks Accept or Reject
3. Export reviewed results back to JSON → sync to DB

**Best for:** High-volume quick grading, non-linguist reviewers, "spot check" passes
**Hosting:** Docker (very simple setup)
**API:** Simple JSON (low integration effort)

---

### Tool Comparison

| Tool | Type | Best For | Integration Effort | Review Speed |
|------|------|----------|-------------------|-------------|
| **MateCAT** | CAT Tool | Professional linguists; detailed post-editing | Moderate (REST API) | Thorough |
| **Weblate** | TMS | Dev teams; Git-native; continuous localization | Low (Git) | Medium |
| **doccano** | Annotator | Quick pass/fail on LLM output | Low (JSON) | Fast |
| **OmegaT** | Desktop CAT | Offline review; industry standard | High (file-based) | Slow (offline) |

---

## The Escalation Connector Module

The C# service that manages HITL integration:

### Trigger Logic
```
if (SemanticScore < 0.85 AND IdiomAudit != "VALID")
OR (RetryCount >= MaxRetries)
    → EscalationHandler.CreateReviewJob(unit)
```

### Data Payload Sent to Review Tool
```json
{
  "pathId": "1,0,2",
  "sourceText": "Configure the middleware pipeline using the [[M_0]] extension method.",
  "arabicDraft": "قم بتكوين خط أنابيب [[M_0]] باستخدام طريقة الامتداد.",
  "context": "section.docs > div.warning > p",
  "semanticScore": 0.74,
  "reviewComment": "Score below threshold. Possible idiom issue with 'middleware pipeline' translation."
}
```

> **Pro Tip:** Include the LLM's reasoning from the Idiom Audit (if it ran) in the `reviewComment`. Reviewers work significantly faster when they understand *why* the system was unsure.

### Callback Handler
The connector supports two synchronization modes:

| Mode | How It Works |
|------|-------------|
| **Webhook** | Review tool calls back to C# endpoint when status = Approved |
| **Polling** | C# service polls the API every N minutes for `Status == 'Approved'` |

### On Approval — DB Update
```sql
UPDATE TranslationUnit SET
    ArabicMD          = '{corrected_arabic_from_reviewer}',
    IsVerified        = 1,
    VerificationLog   = 'Human approved via MateCAT — reviewer: {reviewer_id}',
    VerificationStatus = 'Passed'
WHERE PathID = '1,0,2';
```

---

## Success Criteria

- All Yellow/Red units are either resolved by the Idiom Audit or routed to a human reviewer
- No unresolved Yellow/Red unit reaches Phase H
- Human corrections sync back to DB within the polling interval (or immediately via webhook)
- `IsVerified = 1` is set only after human approval or successful Idiom Audit

---

## 💡 Architect Suggestions

**1. Prioritize HITL queue by semantic score, not arrival order**
The lowest-scoring Red units represent the highest risk. Sort the review queue by `SemanticScore ASC` so reviewers address the worst cases first. If the review team runs out of time, at least the most dangerous translations have been corrected.

**2. Add LLM reasoning to the reviewer's comments field**
When the Idiom Audit ran and returned `REDO`, include its one-line reason in the review tool comment. Example: `"LLM audit: Arabic phrase 'خط أنابيب' is technically correct but context suggests 'وسيط' is preferred."` This halves review time.

**3. Build a "batch pre-approve" flow for known-good Yellow patterns**
After running HITL for a while, you will see recurring Yellow patterns that are always approved (certain common idioms). Build a rule-based pre-approver: if the Yellow unit matches a known-good pattern from previous reviews, auto-approve it without routing to human. Log it as `'Auto-approved: known idiom pattern'`.

**4. Separate reviewer pools by content type**
Technical documentation requires a technical reviewer (developer + Arabic). Marketing content requires a native-speaker copywriter. doccano supports multiple reviewer groups — assign content clusters from Phase C to the appropriate reviewer pool.

**5. Track HITL rate per page cluster as a pipeline health metric**
If a specific cluster (e.g., the Authentication cluster) consistently has 30% HITL escalation, it means the glossary or prompt style rules for that cluster are insufficient. Use this data to improve Phase C and Phase D before the next translation run.

**6. Store reviewer edits as training signal**
Every human correction is a gold-standard EN→AR pair. After accumulating 500+ corrections, fine-tune a small translation model or at minimum add the corrections to the Translation Memory as highest-priority entries for Phase D caching.
