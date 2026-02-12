This refined specification focuses on the **Structural Mapping** architecture. It ensures that the translation is context-aware and that the final Arabic page remains interactive by linking back to the original English source via annotations.

---

## Technical Specification: Semantic HTML Translation & Annotation System

### 1. Objective

To build a C# system using **AngleSharp** that extracts translatable content from English HTML, preserves semantic metadata (tags/classes) for LLM context, and later reconstructs a localized Arabic version. The system must support **asynchronous restoration** (extraction and restoration happen in different sessions) and include a **Source Annotation** feature (hover over Arabic to see English).

### 2. Core Architecture: The Structural Path Logic

To avoid state-dependency, we use a **Numerical Path Index**. Every translatable element is identified by its "coordinate" in the DOM tree (e.g., `0,1,5,2`).

* **Extraction:** Analyzes the English DOM, generates paths, and converts content to Markdown.
* **Storage:** Saves the Path, Semantic Context, and English Markdown to a DB.
* **Restoration:** Loads the English HTML, fetches Arabic translations, and uses the Path to inject content and tooltips.

---

### 3. Phase I: Extraction & Semantic Mapping

#### A. Unit Identification

The developer must identify "Translation Units" based on these rules:

* **Mixed Content Blocks:** Elements like `<p>`, `<li>`, `<td>`, or `<section>` that contain text mixed with inline formatting (`<b>`, `<a>`, `<span>`). Treat the entire `InnerHtml` as one unit.
* **Standalone Attributes:** Specifically `alt`, `title`, and `placeholder`.

#### B. Semantic Metadata (The LLM Context)

For every unit, capture a "Context String" to help the LLM understand the tone:

* **Ancestry:** Capture the tag name and CSS classes of parents up to 4 levels high.
* **Example:** `section.main > div.author > p.bio`. This informs the LLM to use "Biographical/Formal" Arabic rather than "Technical" Arabic.

#### C. Formatting

* Convert `InnerHtml` to **Markdown** (using `ReverseMarkdown`). This preserves bold/italic/links in a format LLMs natively understand.

---

### 4. Phase II: Database Schema

The developer should implement a storage model with these specific fields:

| Field | Type | Description |
| --- | --- | --- |
| `PathID` | `string` | The stable coordinate (e.g., `0,1,12,3`). |
| `Context` | `string` | The semantic breadcrumb (Tags + Classes). |
| `EnglishMD` | `string` | The source Markdown content. |
| `ArabicMD` | `string` | The translated Markdown (to be filled by LLM). |

---

### 5. Phase III: Restoration & Annotation

The restoration phase must be able to run independently of the extraction process.

#### A. Node Targeting & Injection

1. Load the English HTML string into a new AngleSharp `IDocument`.
2. Iterate through the `ArabicMD` records.
3. Traverse the DOM using the `PathID` (e.g., `doc.Root.Children[1].Children[12]...`).
4. Convert `ArabicMD` to HTML and inject into `InnerHtml`.

#### B. English Source Annotation (Hover Feature)

To facilitate the "Mouse Over" requirement:

1. Wrap the translated content (or the parent element) in a container or use the `title` attribute.
2. **Recommended:** Store the `EnglishMD` (converted back to plain text) in a `data-source-en` attribute on the element.
3. **UI Implementation:** ```html
<p class="ar-content" title="Original: [English Text Here]">...الخدمات المصغرة...</p>
```


```



#### C. RTL & Localization Flip

* Set `dir="rtl"` and `lang="ar"` on the `<html>` tag.
* Update any directional attributes (e.g., changing an "arrow-right" class to "arrow-left" if applicable).

---

### 6. Developer Logic Constraints

* **Statelessness:** The system must not rely on keeping a "live" DOM object in memory between extraction and restoration. The `PathID` is the only bridge.
* **Structural Integrity:** When injecting Arabic HTML, ensure it doesn't break the parent container's layout. AngleSharp’s `.InnerHtml` setter is mandatory here to ensure proper parsing of the new content.
* **Markdown Preservation:** The developer must ensure the LLM does not translate Markdown syntax (like `[]()` or `**`).

---

### 7. Success Criteria

* The developer can recreate the entire page in Arabic using only the original English HTML and the database records.
* Hovering over a translated paragraph reveals the original English text.
* Semantic classes (like `author` or `side-bar`) are used in the LLM prompt to improve translation accuracy.

**Would you like me to provide a sample JSON payload representing exactly what should be sent to the LLM including this semantic metadata?**
