---
name: youtube-keywords
description: |
  Extract hierarchical SEO search autocomplete trees from YouTube via public suggest service without API keys.
  Use when: (1) Mining audience search intent and phrasing, (2) Generating L1/L2/L3 keyword trees for video scripts and packaging,
  (3) Uncovering common viewer misconceptions, questions, and pain points for research briefs, (4) Formulating 3 anchored search seeds.
allowed-tools: Bash, Read, Write
---

# YouTube Keywords Tool (`youtube_keywords`)

The `youtube_keywords` tool queries YouTube's public search autocomplete endpoint (`suggestqueries.google.com`) to extract real search demand trees without requiring API keys or third-party quotas.

It operates strictly in a **safe, single-threaded mode** with randomized delays (1.5–5.0s jitter) to protect the production environment from rate limits (HTTP 429).

---

## 1. Semantic Autocomplete Stem Generator (The 3-Angle Root Architecture)

When initiating autocomplete keyword extraction, the agent **MUST** generate exactly three (3) divergent root seeds. 

### Mental Model & Autocomplete Mechanics
Seeds are **NOT** final script tags, titles, or completed SEO keywords. They are **divergent entry points** for the YouTube Autocomplete API. 
* YouTube predicts queries by matching prefix stems. Every added word acts as a filter that permanently cuts off downstream suggestion branches.
* If you over-specify or add non-essential words, you kill high-volume search queries before the API can return them.

### Operational Rules

1. **The Minimum Viable Anchor Rule (1–3 Words Ceiling):**
   * Keep only the words strictly necessary to lock the query into the target Knowledge Graph entity.
   * **1 word:** Allowed ONLY if the term is globally unique and unambiguous across all industries (e.g., `sciatica`, `cordyceps`, `401k`).
   * **2–3 words:** Standard for all other topics where context is required (e.g., `mesh wifi`, `heater core`).
   * **Hard Ceiling:** Never exceed 3 content words. Queries with 4+ words kill autocomplete output.

2. **The Disambiguation & Pruning Test:**
   * **When to DROP a word (Pruning):** If deleting a word leaves the query 100% inside the target topic, delete it immediately. Ban artificial filler nouns (`event`, `process`, `system`, `method`, `overview`, `concept`) and conversational prefixes (`how to`, `why`, `best`, `guide`).
   * **When to KEEP a word (Disambiguation):** YouTube Autocomplete is stateless. If a mechanism or symptom word is polysemic (exists across multiple unrelated industries, e.g., *wear tear*, *ground loop*, *dead zones*), you MUST retain a 1-word Domain Anchor (e.g., `rental wear tear`, `audio ground loop`, `wifi dead zones`).

3. **Angle Separation (Intent Divergence over Blind Vocabulary Ban):**
   * **Seed 1 (Layman / Phenomenon):** How a non-expert queries the observable symptom, pain point, or curiosity.
   * **Seed 2 (Canonical Subject):** The recognized industry, legal, technical, or encyclopedic name of the subject.
   * **Seed 3 (Mechanism / Catalyst):** The specific technical trigger, failing component, statutory rule, or origin factor.
   * **Disambiguation Exception for Shared Words:** Seeds must NOT be simple restatements or synonyms of each other. However, retaining the same single Domain Anchor word (e.g., `tax`, `wifi`, `audio`) across seeds is **mandatory** whenever omitting it causes the seed to match unrelated industries.

### Output Format
```text
Seed 1: [layman / phenomenon stem]
Seed 2: [canonical subject stem]
Seed 3: [mechanism / catalyst stem]
```

### Universal Multi-Domain Benchmarks

| Domain | Core Topic | Seed 1 (Layman / Phenomenon) | Seed 2 (Canonical Subject) | Seed 3 (Mechanism / Catalyst) |
|---|---|---|---|---|
| **Science** | Why Dinosaurs Went Extinct | `dinosaur extinction` | `cretaceous period` | `chicxulub asteroid` |
| **History** | Origin of Money & Currency | `before money` | `currency evolution` | `ancient barter` |
| **Legal / Tax** | UK Inheritance Tax Rules | `gift property uk` | `inheritance tax uk` | `seven year rule` |
| **Automotive** | Clogged Heater Core Flush | `car heater cold` | `heater core flush` | `heater core backflush` |
| **Hardware / Tech** | Fixing Home Wi-Fi Dead Zones | `boost wifi signal` | `mesh wifi setup` | `wifi dead zones` |
| **Biology / Nature** | Ophiocordyceps Zombie Ants | `zombie ant` | `cordyceps fungus` | `ant parasite` |

---

## 2. Tool Contract & Interface

- **Tool Tier:** `ANALYZE`
- **Capability:** `analysis`
- **Runtime:** `LOCAL` (Standard Python `urllib`, no external dependencies)
- **Module:** `tools.analysis.youtube_keywords`
- **Class:** `YouTubeKeywords`

### Input Arguments

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `queries` | `array[str]` | Optional* | - | List of 1 to 3 seed phrases generated per the Stem Generator rules. |
| `query` | `string` | Optional* | - | Single seed phrase or comma-separated list of up to 3 seeds. |
| `output_path` | `string` | No | `None` | Path to persist the resulting `keywords.json` file on disk. |
| `hl` | `string` | No | `"en"` | Language code ISO 639-1 (pass only if non-English from `target_market`). |
| `gl` | `string` | No | `""` | Country/region ISO 3166-1 alpha-2 code (pass only if region-specific from `target_market`; default is unpinned global). |
| `source` | `string` | No | `"combined"` | Search source: `"combined"` (default: dual-pass YouTube + unique Google Web). Optional overrides: `"youtube"`, `"google_web"`. |

*\* Either `queries` or `query` must be provided.*

### Output Schema (`ToolResult.data`)

```json
{
  "primary_seed": "primary topic term",
  "semantic_seeds": [
    "canonical term",
    "mechanism term"
  ],
  "level_2_queries_yt": [
    "yt l2 query 1",
    "yt l2 query 2"
  ],
  "level_2_queries_web": [
    "web l2 query 1 (unique)",
    "web l2 query 2 (unique)"
  ],
  "level_3_long_tail_yt": [
    "yt l3 query 1",
    "yt l3 query 2"
  ],
  "level_3_long_tail_web": [
    "web l3 query 1 (unique)",
    "web l3 query 2 (unique)"
  ],
  "source": "combined",
  "language": "en",
  "region": "",
  "collected_at": "2026-09-10T18:00:00Z"
}
```

---

## 3. Agent Contextual Filtering Protocol

Autocomplete reflects raw keystrokes from real users, which may include homonyms, temporary memes, or unrelated transactional noise.

The agent **MUST** apply this contextual filter **immediately upon collection** before utilizing queries for topic investigation or packaging into final brief metadata:

1. **Platform Nature Understanding:**
   - `level_2_queries_yt` & `level_3_long_tail_yt`: Reflect what real viewers type on the target video platform (YouTube) when searching specifically for video content, explainers, or visual demonstrations.
   - `level_2_queries_web` & `level_3_long_tail_web`: Reflect what users search for across the open web when seeking general information, definitions, news, articles, or answering broad questions (not limited to video).
2. **Topical Relevance Gate:** Discard suggestions that stray outside the core subject (e.g. rap songs, entertainment titles, toys, unrelated celebrities, foreign jurisdictions).
3. **Temporal Validity Gate:**
   - For timely/current topics: Discard suggestions citing outdated years. Retain the current year (`2026`) or timeless queries.
   - For historical topics: Retain relevant historical dates matching the narrative.
4. **Intent Gate:**
   - Remove purely transactional noise (e.g. `login`, `portal sign in`, `pdf download`, `apk`, `phone number`) unless the video specifically covers paperwork submission.
   - Remove file/template asset downloads (e.g. `excel`, `spreadsheet`, `.xlsx`, `template`, `software`).
   - Retain computational queries (e.g. `calculator`, `calculation`, `how to calculate`, `formula`, `how much`): they signal a dedicated on-screen numerical breakdown segment for the script.
5. **Short Root Absorption Gate:** Discard a short 2–3 word query if it appears verbatim as the root of multiple longer, more specific queries already present in the list (e.g., discard bare `unrealised gains` if `what are unrealised gains` or `unrealised capital gains tax` exists; discard `mesh wifi` if `mesh wifi setup guide` exists).
6. **Syntactic Duplicate Gate:** Discard trivial word reorderings / permutations of the exact same words conveying identical intent (e.g., discard `does your super get taxed australia` if `is super taxed in australia` is present; keep only one). Do not discard genuine lexical or technical variations.

---

## 4. Lifecycle & Workflow (Collection -> Scratch Sanitization -> Brief Assembly)

Autocomplete data moves through three phases:

1. **Collection & In-Place Sanitization:**
   - The tool writes raw autocomplete trees to `projects/<project_id>/keywords.json`.
   - The agent reads `keywords.json`, applies the Section 3 filtering gates directly, and overwrites `projects/<project_id>/keywords.json` with the cleaned query list.

2. **Upstream Research:**
   - The agent uses the cleaned queries to discover real audience search questions, misconceptions, and metric benchmarks during topic research.

3. **Downstream Packaging:**
   - Cleaned keywords are packaged into `research_brief.metadata.seo_keywords` as the validated SEO keyword bundle for the project.

---

## 5. Usage Example (Python)

```python
from tools.analysis.youtube_keywords import YouTubeKeywords

tool = YouTubeKeywords()
result = tool.execute({
    "queries": [
        "boost wifi signal",
        "mesh wifi setup",
        "wifi dead zones"
    ],
    "gl": "",
    "hl": "en",
    "source": "combined",
    "output_path": "projects/mesh_explainer/keywords.json"
})

if result.success:
    keywords = result.data
    # YouTube-specific video queries: keywords["level_2_queries_yt"], keywords["level_3_long_tail_yt"]
    # Net-new Web search queries: keywords["level_2_queries_web"], keywords["level_3_long_tail_web"]
```

---

## 6. Error Handling & Fallback Strategy

| Condition / Error | Cause | Expected Tool & Agent Action |
|---|---|---|
| **HTTP 429 (Too Many Requests)** | Autocomplete rate limits triggered by burst requests. | Tool automatically backs off 30s + jitter and retries. If retries fail, returns `success: false`. The calling agent **must not crash**: log a warning, fall back to standard web search, and note `Keywords: Fallback (HTTP 429)` in the stage report. |
| **Empty Suggestions (`[]`)** | Seed phrase is overly long, obscure, or contains rare tokens. | Tool skips further expansion for that seed without failing. The agent proceeds with remaining seeds, or tries simplifying the seed to a tighter 1-2 word stem. |
| **Network Timeout / Connection Error** | Connectivity interruption to suggest endpoint. | Tool waits 25–30s and retries up to 2 times. If failure persists, returns `success: false`. The calling agent proceeds with standard web search. |
