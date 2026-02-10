## Developer Task: Protection & Verification Module

### 1. Objective

To build a **Pre-processing Masking Engine** and a **Post-translation Audit System**. This module ensures that technical tokens (CLI, code, paths) are never corrupted by the LLM and are 100% verified before being stored for restoration.

---

### 2. Phase I: The Masking Engine (Pre-Extraction)

Before sending any `TranslationUnit` to the DB/LLM, the developer must identify and "shield" protected blocks.

#### A. Identification Criteria (The "Protect" List)

The engine must scan `InnerHtml` and `Attributes` for:

* **Explicit Code:** Content inside `<code>`, `<pre>`, `<kbd>`.
* **CLI Patterns:** Commands starting with `$`, `>`, `sudo`, `npm`, `git`, `docker`.
* **Technical Parameters:** Any string matching `\B--[a-z0-9-]+` or ` \-[a-z]`.
* **File Infrastructure:** Paths matching `(?:\/|\\|[a-z]:\\)[^\s\n]+` and Environment Variables (e.g., `$PATH`, `{{secret}}`).
* **Branding/Proper Nouns:** A dictionary-based list (e.g., `AngleSharp`, `FastText`).

#### B. The Masking Process

1. Generate a unique token for each match: `[[M_0]]`, `[[M_1]]`, etc.
2. Replace the original text with the token.
3. Store the mapping in a **Protection Manifest** (JSON) linked to the `PathID`.

---

### 3. Phase II: The Verification Manifest (Database)

The database must be extended to store the "Source of Truth" for protected blocks. This is critical for the "Separate Time" restoration requirement.

**Required Schema Fields:**

* `PathID`: The structural path index.
* `SourceMD`: The English Markdown with placeholders.
* `ProtectionManifest`: A JSON object (e.g., `{"[[M_0]]": "sudo apt-get", "[[M_1]]": "/var/www"}`).
* `VerificationStatus`: `Pending`, `Passed`, or `Failed`.

---

### 4. Phase III: Post-Translation Audit

Once the Arabic translation returns, an automated script must verify the integrity of the placeholders before the unit is approved for the site.

#### Verification Rules:

1. **Parity Check:** Count of `[[M_x]]` in English must exactly match the count in Arabic.
2. **Corruption Check:** Ensure the LLM didn't alter the placeholder (e.g., adding spaces `[[ M_0 ]]` or translating to Arabic characters `[[م_0]]`).
3. **Literal Match:** Confirm that no original protected strings (e.g., "sudo") appear in the raw Arabic output outside of their placeholders (this detects "Leaked" translations).

---

### 5. Phase IV: Unmasking & Annotation (Restoration)

During the separate Restoration Phase:

1. **Fetch:** Load the English HTML and the corresponding Arabic DB record.
2. **Inject:** Replace `[[M_x]]` tokens in the Arabic string with the raw values from the `ProtectionManifest`.
3. **Annotate:** Wrap the restored tokens in a `<span class="source-hint">` or add a `title` attribute to the parent containing the original English text for the "Mouse Over" feature.

---

### 6. Technical Implementation Guidance

* **Regex Performance:** Use compiled `Regex` objects for the pattern matching.
* **Statelessness:** Ensure the `ProtectionManifest` contains everything needed to restore the string; do not rely on the original HTML being "loaded" in memory during the Audit.
* **Language-Specific Edge Cases:** Ensure the masking engine accounts for Arabic punctuation (which might wrap around a placeholder differently than English).

---

### 7. Success Metrics

* **0% Corruption:** No CLI command or Path is altered in the final Arabic output.
* **Traceability:** Every Arabic sentence can be traced back to its specific English protected blocks via the `PathID`.
* **Automation:** The Audit system flags any LLM hallucinations (deleted or modified tokens) without human intervention.

**Would you like me to create a set of "Failure Samples" (corrupted LLM outputs) for the developer to use as unit tests for the Audit phase?**
