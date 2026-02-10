# Intelligent Pre-Translation Text Mining Techniques

**Goal:** Extract glossary candidates, fixed content, and technical terminology from a static website before LLM translation using proper text mining—not regex hacking.

---

## 1. Hyperlinks as Glossary Indicators

**Concept:**  
Internal links often point to important concepts, definitions, or technical terms. If an author repeatedly links the same term across multiple pages, it's a strong signal that this term needs consistent translation.

**What to extract:**
- Link anchor text (the clickable text)
- Link target (the definition/concept page)
- Link frequency across the site
- Pages where the term is linked from

**Example:**

```
Page: /docs/getting-started.html
Link: <a href="/concepts/dependency-injection.html">dependency injection</a>

Page: /docs/testing.html  
Link: <a href="/concepts/dependency-injection.html">dependency injection</a>

Page: /tutorials/advanced.html
Link: <a href="/concepts/dependency-injection.html">DI container</a>
```

**Analysis result:**
- Term: "dependency injection" / "DI container"
- Target concept: `/concepts/dependency-injection.html`
- Frequency: 3 links from different sections
- **Glossary candidate: HIGH** (cross-referenced, has canonical definition page)

**Benefits:**
- Author already did the curation work
- Semantic relationships preserved (term → definition page)
- Variant detection (different ways to refer to same concept)
- Context-aware (see how term is used in different sections)

---

## 2. TF-IDF (Term Frequency - Inverse Document Frequency)

**Concept:**  
Identify terms that are distinctive to specific pages or sections, not just common everywhere. High TF-IDF score = term is important to *this* document but rare across the site.

**Formula:**
- **TF** (Term Frequency): How often does the term appear in this page?
- **IDF** (Inverse Document Frequency): How rare is this term across all pages?
- **TF-IDF = TF × IDF**

**What it finds:**
- Technical terminology specific to each section
- Filters out generic words like "function", "method", "example"
- Reveals page/chapter themes

**Example:**

**Page A: "Introduction to ASP.NET"**
- Terms with high TF-IDF: "middleware", "pipeline", "HttpContext", "Kestrel"
- Low TF-IDF: "application", "server", "request" (appear everywhere)

**Page B: "Entity Framework Core"**  
- Terms with high TF-IDF: "DbContext", "migration", "LINQ", "EF Core"
- Low TF-IDF: "database", "query", "entity" (common across database pages)

**Glossary extraction:**
- Collect top 10-20 TF-IDF terms per page
- Terms appearing in top 10 of multiple pages = core technical vocabulary
- Terms unique to one page = section-specific concepts

**Benefits:**
- Automatic noise filtering (no "the", "is", "and" nonsense)
- Finds what makes each page unique
- Language-agnostic (works for any language)

---

## 3. Document Clustering

**Concept:**  
Group similar pages together based on content, then extract shared terminology per cluster. Pages in the same cluster discuss related topics and likely share technical vocabulary.

**Clustering methods:**
- K-means clustering on TF-IDF vectors
- Hierarchical clustering
- Topic modeling (LDA - Latent Dirichlet Allocation)

**What it reveals:**

**Example site structure after clustering:**

**Cluster 1: "Authentication & Security" (8 pages)**
- Shared terms: "JWT", "OAuth", "bearer token", "authentication", "authorization", "claim"
- Glossary: These terms must be translated consistently across all 8 pages

**Cluster 2: "Database Operations" (12 pages)**
- Shared terms: "Entity Framework", "DbContext", "migration", "LINQ query", "connection string"
- Glossary: Consistent translation needed for database terminology

**Cluster 3: "API Development" (15 pages)**
- Shared terms: "endpoint", "controller", "routing", "middleware", "HTTP verb"
- Glossary: API-specific vocabulary set

**Cluster 4: "Testing" (6 pages)**
- Shared terms: "unit test", "integration test", "mock", "assertion", "test fixture"
- Glossary: Testing terminology

**Benefits:**
- Automatic semantic grouping
- Identifies domain-specific vocabularies
- Reveals site structure and topic boundaries
- Reduces glossary overlap (different meanings in different contexts)

---

## 4. Named Entity Recognition (NER)

**Concept:**  
Automatically identify proper nouns, technologies, products, companies, and frameworks that should typically remain untranslated or need special handling.

**Entity types to detect:**
- **Technologies:** .NET, React, PostgreSQL, Kubernetes
- **Products:** Visual Studio, Azure, GitHub
- **Companies:** Microsoft, Google, Amazon
- **Frameworks:** Entity Framework, SignalR, ASP.NET Core
- **Programming languages:** C#, Python, JavaScript
- **Standards/Protocols:** HTTP, REST, OAuth, SSL/TLS

**Example text:**

```
"To deploy your ASP.NET Core application to Azure App Service, 
you'll need to configure the Azure CLI and ensure your 
DbContext is compatible with Azure SQL Database."
```

**NER extraction:**
- `ASP.NET Core` → Technology/Framework
- `Azure App Service` → Product/Platform
- `Azure CLI` → Tool
- `DbContext` → Technical term
- `Azure SQL Database` → Product

**Glossary decisions:**
- Technologies/Products → **Keep in English** (globally recognized)
- Technical terms like `DbContext` → **Transliterate or keep** depending on target language conventions
- Mix with TF-IDF: If `DbContext` appears 50+ times, definitely add to glossary

**Benefits:**
- Prevents incorrect translation of proper nouns
- Identifies industry-standard terms
- Reveals technology stack of the documentation
- Helps decide translation vs. transliteration strategy

---

## 5. Co-occurrence Analysis (Collocation Detection)

**Concept:**  
Find terms that frequently appear together and should be treated as single units. Multi-word technical terms are often more meaningful than individual words.

**What to detect:**
- Bigrams (2-word phrases): "connection string", "design pattern"
- Trigrams (3-word phrases): "dependency injection container", "object relational mapper"
- N-grams with high co-occurrence scores

**Statistical measures:**
- **Pointwise Mutual Information (PMI)**: How much more likely are these words to appear together vs. separately?
- **Log-likelihood ratio**: Statistical significance of co-occurrence
- **Chi-square test**: Independence test for word pairs

**Example:**

**Text corpus analysis:**

```
"dependency" + "injection" appear together: 45 times
"dependency" alone: 120 times  
"injection" alone: 50 times

PMI score: High → these words form a meaningful unit
```

**Detected collocations:**
- "dependency injection" (not "dependency" + "injection")
- "connection string" (not "connection" + "string")
- "design pattern" (not "design" + "pattern")
- "unit test" (not "unit" + "test")
- "middleware pipeline" (not "middleware" + "pipeline")

**Glossary impact:**

**Without collocation detection:**
- Translate "dependency" as "تبعية"
- Translate "injection" as "حقن"
- Result: "تبعية حقن" (meaningless in Arabic technical context)

**With collocation detection:**
- Treat "dependency injection" as single term
- Translate as "حقن الاعتماديات" or keep as "Dependency Injection"
- Consistent across all 45 occurrences

**Benefits:**
- Preserves multi-word technical terms
- Improves translation quality dramatically
- Reduces glossary size (fewer entries, better organized)
- Matches how developers actually use terminology

---

## 6. Link Graph Analysis (PageRank-style)

**Concept:**  
Use internal linking structure to identify the most important concept pages. Pages with many incoming links are central to the documentation and their terminology is critical.

**Graph metrics:**
- **In-degree**: How many pages link to this page?
- **Out-degree**: How many links does this page contain?
- **PageRank score**: Weighted importance based on link structure
- **Betweenness centrality**: How often is this page on the shortest path between other pages?

**Example link graph:**

```
/concepts/dependency-injection.html
  ← Linked from: 23 pages
  → Links to: 5 pages
  PageRank: 0.087 (HIGH)
  
/concepts/middleware.html
  ← Linked from: 18 pages
  → Links to: 7 pages
  PageRank: 0.065 (HIGH)
  
/tutorials/hello-world.html
  ← Linked from: 2 pages
  → Links to: 15 pages
  PageRank: 0.012 (LOW)
  
/reference/api-spec.html
  ← Linked from: 45 pages
  → Links to: 0 pages
  PageRank: 0.123 (VERY HIGH - reference page)
```

**Glossary priority ranking:**

1. **Critical terms** (PageRank > 0.08): Must have perfect translation
   - Terms from: dependency-injection, middleware, api-spec pages
   
2. **Important terms** (PageRank 0.04-0.08): High consistency needed
   - Terms from: authentication, routing, configuration pages
   
3. **Supporting terms** (PageRank < 0.04): Standard translation
   - Terms from: tutorial, example, getting-started pages

**Benefits:**
- Automatic prioritization of glossary entries
- Identifies canonical definition pages
- Reveals documentation structure and concept hierarchy
- Focus translation effort where it matters most

---

## 7. Cross-Reference Detection

**Concept:**  
Identify terms that are explicitly defined somewhere in the documentation, then find all places where those terms are used without definition.

**Pattern matching:**
- Definition indicators: "is defined as", "refers to", "means", section titled "Definitions"
- Reference indicators: "see also", "as discussed in", "refer to"

**Example:**

**Definition page:** `/concepts/middleware.html`
```
"Middleware is software that's assembled into an application pipeline 
to handle requests and responses."
```

**Usage pages:**
- `/tutorials/custom-middleware.html` → mentions "middleware" 12 times
- `/docs/pipeline.html` → mentions "middleware" 8 times  
- `/guides/authentication.html` → mentions "middleware" 5 times

**Glossary entry:**
```
Term: middleware
Definition source: /concepts/middleware.html
Occurrences: 25 times across 3 pages
Translation priority: HIGH (has canonical definition + multiple references)
Proposed translation: برمجيات وسيطة or keep as "Middleware"
```

**Benefits:**
- Links terms to their authoritative definitions
- Ensures consistency between definition and usage
- Provides context for translators
- Validates glossary completeness (every defined term should be in glossary)

---

## 8. Fixed/Repeated Content Detection (Hash-based)

**Concept:**  
Find exact or near-exact duplicate content blocks that appear across multiple pages. Translate once, reuse everywhere.

**Detection methods:**
- **Exact matching**: SHA256 hash of normalized text
- **Fuzzy matching**: Similarity scores (Levenshtein distance, Jaccard similarity)
- **Structural matching**: Same HTML structure with different content

**What to find:**
- Headers and footers
- Navigation menus
- Copyright notices
- Disclaimers and warnings
- Call-to-action blocks
- Code snippets (exact duplicates)

**Example:**

**Repeated disclaimer (appears on 47 pages):**
```
"This feature is currently in preview and subject to change. 
Production use is not recommended."
```

**Hash:** `a7f3c9e2...`  
**Occurrences:** 47 pages  
**Translation strategy:** Translate once, store in translation memory, reuse across all 47 pages  
**Cost savings:** 46 fewer translation API calls

**Repeated navigation (appears on all 120 pages):**
```
Home | Documentation | API Reference | Tutorials | Community
```

**Translation strategy:** Translate once, apply globally

**Benefits:**
- Massive cost reduction (translate once, use many times)
- Perfect consistency for repeated UI elements
- Fast translation for common blocks
- Easy maintenance (update translation in one place)

---

## Combined Workflow

**Phase 1: Content Ingestion**
1. Parse static site into database
2. Extract all text blocks, links, structure

**Phase 2: Text Mining Analysis**
1. Run hyperlink analysis → build term→definition mappings
2. Calculate TF-IDF scores → identify distinctive terms per page
3. Perform document clustering → group related pages, extract shared vocabulary
4. Apply NER → identify technologies, products, proper nouns
5. Detect collocations → find multi-word technical terms
6. Analyze link graph → rank page importance, prioritize concepts
7. Cross-reference detection → map terms to definitions
8. Hash-based duplicate detection → identify reusable content

**Phase 3: Glossary Construction**
1. Merge results from all techniques
2. Score each term by: frequency, importance, cross-references, TF-IDF
3. Human review of top 200-500 terms
4. Finalize translation decisions per term

**Phase 4: Translation**
1. Use glossary + context-aware chunking
2. Apply translation memory for repeated content
3. Validate against glossary
4. Consistency pass

---

## Key Insight

**The 80/20 rule:**
- 20% of terms account for 80% of technical accuracy issues
- These techniques automatically find that critical 20%
- Focus human effort on reviewing high-value glossary entries
- Let algorithms handle the noise filtering

**Bottom line:** Smart text mining before translation saves more effort than any amount of prompt engineering during translation.
