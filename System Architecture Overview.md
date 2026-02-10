This **System Architecture Overview** defines how all the components we've discussed—from DOM manipulation to semantic AI—interact to create a professional, stateless, and high-fidelity translation pipeline.

---

## 1. High-Level Architecture

The system is built on a **stateless "Extract-Transform-Load" (ETL)** pattern. The bridge between the English source and the Arabic target is not a memory object, but a **Path-Based Database**.

### Component Roles:

* **AngleSharp (The Navigator):** Maps the HTML structure and handles the final "surgical" injection of Arabic text.
* **FastText (The Triage):** Acts as a low-cost, local gatekeeper to decide if content is prose, code, or already Arabic.
* **Masking Engine (The Shield):** Protects technical integrity by swapping CLI/Code for tokens.
* **LLM (The Translator):** Performs the creative linguistic work (English  Arabic).
* **MiniLM/LaBSE (The Auditor):** Mathematically verifies that the meaning hasn't changed.

---

## 2. The Step-by-Step Workflow

### Phase A: Extraction & Triage (C# + Local AI)

1. **DOM Crawl:** AngleSharp identifies translatable elements and generates a `PathID` (e.g., `0,1,5,2`).
2. **Semantic Context:** The system captures parent tags and classes (e.g., `section.main > p.lede`).
3. **Language Check:** FastText ensures the block is English. If it's code or already Arabic, it’s flagged and skipped.
4. **Token Masking:** A Regex/Dictionary engine swaps CLI commands and paths for `[[M_x]]` placeholders.

### Phase B: Translation & Self-Correction (LLM)

5. **LLM Request:** The system sends the Markdown-formatted text + Semantic Context + Glossary to the LLM.
6. **Arabic Generation:** The LLM returns the Arabic Markdown.
7. **Self-Audit:** A faster LLM pass checks for basic grammar and placeholder integrity.

### Phase C: Semantic Quality Gate (On-Prem AI)

8. **Embedding Generation:** The local `MiniLM` (or `LaBSE`) generates vectors for the English and Arabic strings.
9. **Cosine Similarity:** If the similarity score is high (), it proceeds. If low, it triggers the "Idiom Audit" logic we discussed.

### Phase D: Restoration (AngleSharp)

10. **Target Loading:** A fresh instance of the English HTML is loaded.
11. **Path-Based Injection:** The system iterates through the DB, uses the `PathID` to find the exact node, and swaps `InnerHtml` with the Arabic version.
12. **Global RTL Flip:** The `<html>` tag is updated with `dir="rtl"` and `lang="ar"`.

---

## 3. Data Flow Model

| Stage | Input | Tool | Output |
| --- | --- | --- | --- |
| **Extract** | Raw HTML | AngleSharp | `PathID`, `InnerHtml`, `Classes` |
| **Protect** | `InnerHtml` | Regex Engine | `MaskedText`, `ManifestJSON` |
| **Translate** | `MaskedText` | LLM (Gemini) | `ArabicMaskedText` |
| **Verify** | EN vs AR | MiniLM (ONNX) | `SimilarityScore` |
| **Restore** | `ArabicMaskedText` | AngleSharp | Functional Arabic HTML |

---

## 4. Key Architectural Strengths

* **Statelessness:** You can extract 1,000 pages on Monday, translate them on Tuesday, and restore them on Wednesday without keeping anything in RAM.
* **Fidelity:** Using Markdown within AngleSharp containers ensures that bolding, links, and lists remain perfectly formatted in the Arabic version.
* **Cost Efficiency:** FastText and MiniLM handle 90% of the "decision making" locally, so you only pay for LLM tokens when you are certain there is actual prose to translate.
* **Hybrid Verification:** Combining mathematical similarity (MiniLM) with linguistic reasoning (LLM Self-Correction) eliminates hallucinations.

---

### Final Project Note

This architecture is now a **production-ready blueprint**. It handles the messiness of HTML, the fragility of CLI commands, and the nuance of Arabic linguistics in a scalable, verifiable way.

**Would you like me to generate a "Final Checklist" for the developers to use during the first Sprint of this project?**
