# Phase B — Protect & Mask

*Pipeline position: Second · Depends on: Phase A (TranslationUnit records) · Feeds into: Phase D (Translate)*

---

## Objective

Before any text reaches the LLM, replace every technical token with a safe placeholder. This guarantees that code commands, paths, URLs, environment variables, and brand names survive translation completely unchanged.

---

## Inputs & Outputs

| | Value |
|--|-------|
| **Input** | `EnglishMD` from `TranslationUnit` records |
| **Output** | Masked text + `ProtectionManifest` (JSON) stored per record |
| **Primary tools** | Compiled Regex, FlashText dictionary engine |

---

## Why Masking Is Critical

The LLM sees natural language. To it, `sudo apt-get install nginx` looks like words — it might "translate" `install` to Arabic, break the flag syntax, or reorder arguments. None of that is acceptable.

The masking approach treats protected content as **opaque tokens** the LLM cannot see or reason about. To the LLM, the text becomes:

```
Run [[M_0]] to install the server, then edit [[M_1]] to configure [[M_2]].
```

The LLM translates the natural language around the tokens and returns them untouched. Phase E then verifies every single token is still present and intact.

---

## What Gets Masked

### Category Reference

| Category | Examples | Detection Method |
|----------|---------|-----------------|
| Explicit code blocks | `<code>`, `<pre>`, `<kbd>` content | HTML element type |
| CLI commands | `sudo`, `npm run`, `git clone`, `docker-compose` | Keyword + prefix pattern |
| CLI flags | `--verbose`, `-x`, `--output=file` | Regex: `\B--[a-z0-9-]+` or ` \-[a-z]` |
| File paths | `/var/www/html`, `C:\Users\app` | Regex: Unix + Windows path patterns |
| URLs | `https://example.com/api/v2` | Regex: URL pattern |
| Email addresses | `admin@example.com` | Regex: email pattern |
| Environment variables | `$PATH`, `{{SECRET_KEY}}`, `%APPDATA%` | Regex: env var patterns |
| Proper nouns / Brands | `AngleSharp`, `FastText`, `.NET`, `SignalR` | Dictionary lookup (FlashText) |
| Version numbers | `v3.2.1`, `Node 20.x` | Regex: semver pattern |

---

## Step-by-Step Process

### Step 1 — Scan for Protected Content
Run all detection patterns against `EnglishMD`. Order matters: run **longer/more specific patterns first** to avoid partial matches.

Recommended scan order:
1. Full code blocks (multi-line ```` ``` ``` ````)
2. Inline code (backtick spans)
3. URLs (before email, to avoid partial matches)
4. Email addresses
5. File paths
6. Environment variables
7. CLI commands + flags
8. Brand/proper noun dictionary (FlashText — O(1) lookup)

### Step 2 — Generate Placeholder Tokens
For each match, generate a sequential token: `[[M_0]]`, `[[M_1]]`, `[[M_2]]`…

Rules for tokens:
- Zero-indexed, sequential within each unit
- Format is **always** `[[M_{integer}]]` — no variations
- Tokens are deterministic per unit (same input → same token order)

### Step 3 — Build the Protection Manifest
Store the mapping as JSON in the `ProtectionManifest` field:

```json
{
  "[[M_0]]": "sudo apt-get install nginx",
  "[[M_1]]": "/etc/nginx/nginx.conf",
  "[[M_2]]": "server_name"
}
```

The manifest is **self-contained**: it contains everything needed to unmask during Phase H restoration, independent of the original HTML.

### Step 4 — Write Masked Text to DB
Update the `TranslationUnit` record:

```sql
UPDATE TranslationUnit SET
    MaskedEnglishMD     = 'Run [[M_0]] to install, then edit [[M_1]]...',
    ProtectionManifest  = '{"[[M_0]]": "sudo apt-get install nginx", ...}',
    VerificationStatus  = 'MaskingComplete'
WHERE PathID = '1,0,2';
```

---

## Database Schema for This Phase

```sql
ALTER TABLE TranslationUnit ADD COLUMN (
    MaskedEnglishMD    TEXT,   -- EnglishMD with [[M_x]] substitutions
    ProtectionManifest TEXT,   -- JSON: token → original value
    VerificationStatus TEXT    -- 'MaskingComplete' after this phase
);
```

---

## Phase II — Post-Translation Audit (Runs After Phase D)

Once the LLM returns `ArabicMD`, this module runs three checks **before** the result is saved:

### Check 1 — Parity Check
Count `[[M_x]]` occurrences in `MaskedEnglishMD` and in `ArabicMD`. They must be **exactly equal**.

```
English: [[M_0]], [[M_1]], [[M_2]]  → count: 3
Arabic:  [[M_0]], [[M_1]], [[M_2]]  → count: 3  ✅
Arabic:  [[M_0]], [[M_1]]           → count: 2  ❌ FAIL
```

### Check 2 — Corruption Check
Ensure no placeholder was altered by the LLM. Common corruption patterns to detect:

| Corruption Type | Example | Detection |
|-----------------|---------|-----------|
| Added spaces | `[[ M_0 ]]` | Regex strict match |
| Translated to Arabic characters | `[[م_0]]` | Non-ASCII inside brackets |
| Reordered tokens | `[[M_1]]` where `[[M_0]]` was expected | Sequence validation |
| Partial deletion | `[M_0]` | Bracket count check |

### Check 3 — Leak Check
Scan the raw Arabic output for any original protected strings appearing **outside** their placeholders. For example, if `sudo` appears as raw Arabic text, it was "leaked" by the LLM.

```
Arabic contains "sudo" outside any [[M_x]] → ❌ FAIL (leaked translation)
```

### Audit Result
- All 3 checks pass → proceed to Phase E validation
- Any check fails → mark `VerificationStatus = 'AuditFailed'`, queue for retry in Phase D

---

## Success Metrics

- **0% corruption**: No CLI command, path, or brand name is altered in the final Arabic output
- **100% parity**: Every masked unit has matching token count before and after translation
- **Traceability**: Every Arabic sentence traces back to specific English protected blocks via `PathID`

---

## 💡 Architect Suggestions

**1. Use FlashText for brand/proper noun dictionary (not Regex)**
[FlashText](https://github.com/vi3k6i5/flashtext) matches thousands of dictionary terms in O(n) time regardless of dictionary size. Regex with 500 brand names becomes extremely slow. Build a `ProperNounDictionary` file (one term per line) and load it into FlashText at startup.

**2. Handle Arabic punctuation wrapping**
Arabic text may wrap punctuation around a placeholder differently than English. The corruption check must account for Arabic characters directly adjacent to `[[M_x]]` tokens. Pre-normalize surrounding whitespace before running the parity check.

**3. Persist manifests even for units with zero protected tokens**
Always write a `ProtectionManifest` (even if it's `{}`). This ensures the restoration phase never has to handle a null manifest — it simplifies the code and prevents null reference errors.

**4. Version your brand dictionary**
Store the `DictionaryVersion` (e.g., a hash of the dictionary file) in the DB. When the brand list is updated and pages are re-translated, you can detect which records used the old dictionary and need re-masking.

**5. Consider masking Arabic numbers separately**
Western numerals (`42`, `3.14`) vs Arabic-Indic numerals (`٤٢`) can cause confusion. Decide upfront which numeral system to use in output and mask accordingly if the source contains mixed forms.

**6. Add a "dry run" masking report**
Before the full extraction run, generate a report: how many tokens per category were detected across the whole site. This helps catch over-masking (brand dictionary too aggressive) and under-masking (missing regex patterns) before translation begins.
