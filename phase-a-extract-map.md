# Phase A — Extract & Map

*Pipeline position: First · Depends on: Raw HTML input · Feeds into: Phase B (Protect & Mask)*

---

## Objective

Extract every translatable piece of text from the English HTML and store it in the database with a stable address (`PathID`) and semantic context. This database record is the only thing needed to reconstruct the Arabic page later — no in-memory state is ever relied upon.

---

## Inputs & Outputs

| | Value |
|--|-------|
| **Input** | Raw English HTML file(s) |
| **Output** | `TranslationUnit` records: `PathID`, `Context`, `EnglishMD` |
| **Primary tool** | AngleSharp (C# DOM parser) |
| **Secondary tool** | ReverseMarkdown (HTML → Markdown converter) |

---

## What Gets Extracted

### Translatable Node Types

| Node Type | Examples | Rule |
|-----------|---------|------|
| Mixed-content blocks | `<p>`, `<li>`, `<td>`, `<section>` | Treat entire `InnerHtml` as one unit |
| Inline-formatted blocks | `<b>`, `<a>`, `<span>` inside a block | Captured as part of their parent unit |
| Standalone attributes | `alt`, `title`, `placeholder` | Extracted individually as their own units |

### What Is Skipped

- Blocks that are already Arabic (detected via FastText language check)
- Pure code blocks (`<code>`, `<pre>`, `<kbd>`) — these go directly to Phase B masking
- Navigation/menu duplicates — caught by hash-based deduplication in Phase C
- Empty or whitespace-only nodes

---

## Step-by-Step Process

### Step 1 — DOM Crawl
AngleSharp loads the HTML and walks the DOM tree. For each element that qualifies as a translation unit, it records the node.

### Step 2 — Generate PathID
Every translatable node is assigned a **numerical DOM coordinate** — its position in the tree expressed as child indices from the root.

```
Example: body → section[1] → div[0] → p[2]
PathID:  "1,0,2"
```

This coordinate is **deterministic and stable** as long as the HTML structure is unchanged. It is the sole bridge between extraction and restoration sessions.

### Step 3 — Capture Context (Semantic Breadcrumb)
For every unit, record the ancestor chain up to **4 levels high**, including tag names and CSS classes:

```
section.main > div.author > p.bio
```

This breadcrumb is sent to the LLM to signal the **tone** required:
- `section.docs > pre` → technical, no creative phrasing
- `section.hero > h1` → marketing, impactful
- `div.author > p.bio` → biographical, formal

### Step 4 — Language Check (FastText Triage)
Before converting to Markdown, run a local **FastText** language detection pass:
- English prose → proceed
- Already Arabic → flag as `SkipArabic`, do not process
- Code / numeric-only → flag as `SkipCode`, send directly to masking
- Mixed → proceed (masking in Phase B will handle the code parts)

FastText runs **locally** (no API cost) and handles 90% of triage decisions.

### Step 5 — Convert to Markdown
Convert `InnerHtml` to **Markdown** using `ReverseMarkdown`. This preserves bold, italic, links, and inline code in a format that LLMs natively understand and are less likely to corrupt.

```html
<!-- Input -->
<p>Use <strong>AngleSharp</strong> to parse the <a href="/docs">DOM tree</a>.</p>

<!-- Output Markdown -->
Use **AngleSharp** to parse the [DOM tree](/docs).
```

### Step 6 — Persist to Database
Write the record to the `TranslationUnit` table:

| Field | Value |
|-------|-------|
| `PathID` | `"1,0,2"` |
| `Context` | `"section.main > div.content > p"` |
| `EnglishMD` | `"Use **AngleSharp** to parse the [DOM tree](/docs)."` |
| `ArabicMD` | *(empty — filled in Phase D)* |
| `VerificationStatus` | `Pending` |

---

## Database Schema (Minimum for This Phase)

```sql
CREATE TABLE TranslationUnit (
    Id              INTEGER PRIMARY KEY,
    PathID          TEXT NOT NULL,        -- DOM coordinate e.g. "1,0,2"
    Context         TEXT,                 -- Ancestor breadcrumb
    EnglishMD       TEXT NOT NULL,        -- Source Markdown
    ArabicMD        TEXT,                 -- Filled in Phase D
    VerificationStatus TEXT DEFAULT 'Pending',
    PageUrl         TEXT,                 -- Optional: source page URL
    BlockHash       TEXT                  -- For duplicate detection (Phase C)
);
```

---

## Success Criteria

- Every visible text block on the page has a corresponding `TranslationUnit` record
- `PathID` correctly locates the node when used for restoration (Phase H)
- `Context` breadcrumb is populated and includes at least 2 ancestor levels
- Standalone attributes (`alt`, `title`, `placeholder`) are captured as separate records
- No code-only blocks or Arabic blocks are included

---

## Common Failure Modes

| Problem | Cause | Fix |
|---------|-------|-----|
| PathID points to wrong node after restore | HTML structure changed between extraction and restore | Inject `data-tid` attribute during extraction as a fallback (see Suggestions) |
| Context is empty | No parent classes exist | Fall back to tag-only context: `body > section > p` |
| Duplicate records for repeated navigation | Same block on every page | Add `BlockHash` deduplication before insert |
| Markdown conversion breaks links | ReverseMarkdown version issue | Validate round-trip: Markdown → HTML → Markdown |

---

## 💡 Architect Suggestions

**1. Inject `data-tid` attributes as a PathID fallback**
Before extraction, inject a unique `data-tid` attribute on every target node:
```html
<p data-tid="1,0,2">Use AngleSharp...</p>
```
Store both the PathID and the `data-tid` in the DB. During restoration (Phase H), try PathID first — if the DOM has changed (e.g., a CMS added an extra wrapper div), fall back to `querySelector('[data-tid="1,0,2"]')`. This makes the pipeline resilient to minor HTML structural changes between extraction and publishing.

**2. Keep the DOM structure identical between extraction and restoration**
The PathID approach only works if the HTML structure is stable. Never run extraction on a development build and restoration on a production build if they have different DOM structures. Pin the HTML source version (e.g., use a git commit hash as `PageVersion` in the DB).

**3. Use FastText `lid.176.bin` model for language detection**
The compressed 176-language model is only ~900KB and runs in microseconds. It correctly detects code blocks as non-prose. Avoid the full `lid.176.ftz` model for this use case — overkill.

**4. Normalize Markdown before storing**
Run a normalization pass on `EnglishMD` before storing: trim whitespace, normalize link syntax, strip zero-width characters. This ensures `BlockHash` (for deduplication) is consistent across pages.

**5. Capture `<meta>` and `<title>` tags as translation units**
Page title, meta description, and Open Graph tags are often forgotten. They are critical for SEO on the Arabic version. Add a separate extraction pass for `<head>` content.
