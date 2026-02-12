# Phase H — Restore, Annotate & Publish

*Pipeline position: Final · Depends on: All IsVerified = true TranslationUnit records · Output: Functional Arabic HTML*

---

## Objective

Take the verified Arabic Markdown from the database, inject it back into a fresh copy of the English HTML at the exact DOM location identified by `PathID`, unmask all `[[M_x]]` tokens, apply RTL settings, and optionally annotate elements with their original English source for hover tooltips.

---

## Inputs & Outputs

| | Value |
|--|-------|
| **Input** | Fresh English HTML + `TranslationUnit` records where `IsVerified = true` |
| **Output** | Fully functional Arabic HTML page |
| **Primary tool** | AngleSharp |

---

## Critical Design Principle

**This phase is fully independent.** It requires only:
1. The original English HTML file
2. The database records

It does **not** require the extraction session to still be running. It does not require any in-memory state from previous phases. You can restore a page one year after it was extracted, on a different server, without any shared context — as long as the HTML structure matches what was extracted.

---

## Step-by-Step Process

### Step 1 — Load a Fresh HTML Instance
Create a new AngleSharp `IDocument` from the **original English HTML string**. Do not reuse any AngleSharp document from Phase A. This fresh instance is the canvas for injection.

```csharp
var document = await context.OpenAsync(req => req.Content(originalHtml));
```

### Step 2 — Query All Verified Records
Fetch all `TranslationUnit` records for this page where `IsVerified = true`, ordered by `PathID`.

### Step 3 — Locate Node via PathID
For each record, traverse the DOM using the `PathID` coordinate:

```csharp
// PathID = "1,0,2" means:
// document.DocumentElement.Children[1].Children[0].Children[2]

IElement node = document.DocumentElement;
foreach (var index in pathId.Split(',').Select(int.Parse))
{
    node = node.Children[index];
}
```

**Fallback strategy (if PathID navigation fails):**
Try the `data-tid` attribute injected during Phase A:
```csharp
var fallbackNode = document.QuerySelector($"[data-tid='{pathId}']");
```
If both fail → log `RestorationFailed` for this unit and skip. Do not crash the entire page restoration.

### Step 4 — Convert ArabicMD → HTML
Convert the stored Markdown back to HTML using a Markdown-to-HTML library:

```csharp
var arabicHtml = MarkdownConverter.ToHtml(unit.ArabicMD);
```

### Step 5 — Unmask Tokens
Replace all `[[M_x]]` placeholders in `arabicHtml` using the `ProtectionManifest`:

```csharp
foreach (var (token, original) in manifest)
{
    arabicHtml = arabicHtml.Replace(token, original);
}
```

Verify no `[[M_x]]` tokens remain in the output. If any remain → the manifest is incomplete → log as error.

### Step 6 — Inject into DOM
Set the `InnerHtml` of the located node to the Arabic HTML. Use AngleSharp's `.InnerHtml` setter (not `.TextContent`) to ensure proper HTML parsing of the new content:

```csharp
node.InnerHtml = arabicHtml;
```

### Step 7 — Add Source Annotation (Optional)
For the hover-to-see-English feature, add a `data-source-en` attribute containing the original English text:

```html
<!-- Result after annotation -->
<p class="ar-content" 
   data-source-en="Use AngleSharp to parse the DOM tree."
   title="Original: Use AngleSharp to parse the DOM tree.">
  استخدم AngleSharp لتحليل شجرة DOM.
</p>
```

```csharp
node.SetAttribute("data-source-en", unit.EnglishMD);
node.SetAttribute("title", $"Original: {unit.EnglishMD}");
```

### Step 8 — Apply RTL & Language Settings
After all units are injected, flip the page to Arabic:

```csharp
// Set document language and direction
document.DocumentElement.SetAttribute("dir", "rtl");
document.DocumentElement.SetAttribute("lang", "ar");

// Update directional CSS classes (if applicable)
// e.g., "arrow-right" → "arrow-left"
// e.g., "text-left" → "text-right"
```

### Step 9 — Output the Final HTML
Serialize the modified AngleSharp document to an HTML string and write to the output file:

```csharp
var arabicPage = document.DocumentElement.OuterHtml;
File.WriteAllText(outputPath, arabicPage);
```

---

## Database Updates

```sql
UPDATE TranslationUnit SET
    RestorationStatus = 'Restored',
    RestoredAt        = CURRENT_TIMESTAMP
WHERE PathID = '1,0,2' AND IsVerified = 1;
```

Log any units that failed restoration separately for investigation.

---

## Restoration Failure Modes

| Failure | Cause | Recovery |
|---------|-------|---------|
| `PathID` navigation returns null | DOM structure changed between extraction and restoration | Use `data-tid` fallback attribute |
| Remaining `[[M_x]]` after unmasking | Manifest is missing a token | Log error, keep English version for that unit |
| `InnerHtml` injection breaks layout | Arabic HTML has unbalanced tags | AngleSharp's setter auto-parses; validate Markdown output |
| Wrong node targeted | Page template changed | Re-run Phase A extraction against current HTML |

---

## Final Output Checklist

- [ ] Every `IsVerified = true` unit is injected into the correct DOM node
- [ ] All `[[M_x]]` tokens are unmasked — no placeholders remain in output HTML
- [ ] `dir="rtl"` and `lang="ar"` are set on `<html>` tag
- [ ] `<title>` and meta tags are translated
- [ ] All links remain functional (href values unchanged)
- [ ] All interactive elements (buttons, forms, inputs) remain functional
- [ ] Source annotation attributes are present on translated elements (if enabled)
- [ ] No English `TranslationUnit` content remains untranslated (check for `IsVerified = false` units)

---

## 💡 Architect Suggestions

**1. Inject `data-tid` during Phase A for PathID fallback resilience**
The most important suggestion for this phase. During Phase A extraction, before saving the record, inject a `data-tid` attribute directly on the source HTML node:
```html
<p data-tid="1,0,2">Original text here</p>
```
Store both `PathID` and `data-tid` in the DB. During restoration, if `PathID` traversal fails (because a CMS added a wrapper div, or a template was updated), `document.QuerySelector('[data-tid="1,0,2"]')` serves as an accurate fallback. This makes the pipeline resilient to minor template changes without requiring a full re-extraction.

**2. Keep the DOM structure identical between extraction and restoration versions**
Pin the HTML source to a specific version (git commit, build hash). Store `HtmlVersion` in the `TranslationUnit` record. At the start of Phase H, verify the source HTML hash matches what was used during Phase A. If it doesn't match, warn the operator before proceeding — a changed DOM structure may cause PathID mismatches on some nodes.

**3. Apply RTL-aware CSS at page level, not just `dir` attribute**
The `dir="rtl"` attribute handles text direction. But some CSS classes are directional (`ml-4`, `text-left`, `float-right`, Bootstrap grid, etc.). During restoration, scan for common directional class names and replace their mirrored equivalents. Maintain a configurable `RTLClassMap`:
```json
{ "text-left": "text-right", "ml-4": "mr-4", "pl-2": "pr-2" }
```

**4. Separate restoration from publishing**
Generate all Arabic HTML files to a staging directory first. Run a full link-integrity check (all internal links resolve, no broken images) before copying to the production directory. Never write directly to production.

**5. Handle `<meta>` and `<title>` restoration separately**
These are in `<head>`, not in the main content DOM. They need separate PathID records (captured in Phase A as suggested) and separate injection logic. They are critical for SEO on the Arabic version.

**6. Use AngleSharp's `.InnerHtml` setter, not `.TextContent`**
`.TextContent` treats the content as plain text and escapes HTML entities. `.InnerHtml` correctly parses the Arabic HTML string and inserts it as DOM nodes. Always use `.InnerHtml` when the translated content may contain links, bold text, or any inline HTML.

**7. Generate a restoration report after each run**
Output a summary: total units, successfully restored, PathID fallbacks used, units skipped (no verified translation), units that failed completely. This report is the quality gate before publishing — a >5% failure rate should trigger investigation before the page goes live.
