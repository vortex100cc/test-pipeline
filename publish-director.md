# Publish Director - Animation-Hybrid Pipeline

## When To Use

Package the animation so the metadata, thumbnail concept, and platform framing reflect the actual visual system of the project.

## Prerequisites

| Layer | Resource | Purpose |
|-------|----------|---------|
| Schema | `schemas/artifacts/publish_log.schema.json` | Artifact validation |
| Prior artifacts | `state.artifacts["compose"]["render_report"]`, `state.artifacts["proposal"]["proposal_packet"]`, `state.artifacts["research"]["research_brief"]`, `state.artifacts["script"]["script"]` | Final outputs and topic framing |
| Playbook | Active style playbook | Visual naming consistency |

## Process

### 1. Match Packaging To The Animation Mode

Examples:

- diagram-heavy videos should look structured and legible,
- kinetic-type pieces should package around strong copy,
- illustrative animation should package around hero imagery.

### 2. Preserve Visual-System Truth

Store in `publish_log.metadata`:

- `animation_mode`
- `hero_frame_notes`
- `thumbnail_concept`
- `platform_notes`

### 3. Thumbnail Asset Generation
1. Formulate the image generation prompt using the `thumbnail_concept` from the proposal.
2. Prepend `playbook.asset_generation.thumbnail_prompt_prefix`.
3. Ensure the render format is strictly **16:9 widescreen (1920x1080 resolution)**.
4. Call the image generation tool to render `projects/<project_id>/assets/thumbnails/thumbnail.png`.
5. Supply the image path to `export_bundle(thumbnail_path=...)`.

### 4. Standard YouTube Description Builder
Assemble the video description using this exact structure:

[One concise, SEO-focused paragraph outlining the premise and conflict. Naturally integrate the YouTube autocomplete keywords loaded from `research_brief.metadata.seo_keywords` (prioritizing `level_2_queries_yt` and `level_3_long_tail_yt`). Do NOT reveal the conclusion or full outcome — viewers must watch to find out.]

Official Sources:
- [Primary Official Act / Statutory Instrument]: [Official government URL from ResearchBrief]
- [Government Guidance / Official Register]: [Official portal URL from ResearchBrief]
*(Note: List strictly 1–3 authoritative official state/parliamentary sources to frame the video as an independent analysis of official records, not a compilation of press articles.)*

[Disclaimer copied verbatim from playbook.disclaimer.text]

### 5. Content History Logging
Append a record of the finished video to the channel's history ledger: `channels/<channel_id>/history.json` (where `<channel_id>` is the active channel subfolder assigned to the video project).
*(Note: The JSON block below is a structural template. Do NOT copy placeholder values verbatim; dynamically populate each field from the current project's metadata):*

```json
{
  "date": "YYYY-MM-DD",
  "title": "Final YouTube Title",
  "topic": "Core topic description",
  "thumbnail_concept": {
    "visual": "Description of persona and layout",
    "text": "WORDS ON THUMBNAIL"
  },
  "hook_concept": "Summary of opening hook tension",
  "is_hot": true,
  "primary_sources": [
    "https://..."
  ],
  "seo_keywords": {
    "primary_seed": "primary topic term",
    "semantic_seeds": [
      "canonical term",
      "mechanism term"
    ],
    "level_2_queries_yt": [
      "yt l2 query 1"
    ],
    "level_2_queries_web": [
      "web l2 query 1"
    ],
    "level_3_long_tail_yt": [
      "yt l3 query 1"
    ],
    "level_3_long_tail_web": [
      "web l3 query 1"
    ]
  }
}
```

### 6. Quality Gate

- metadata fits the actual animation mode,
- thumbnail concept prioritizes high-CTR platform conventions (prominent recognizable persona related to the topic + bold text hook + background/graphical accents). Internal animation style does NOT constrain thumbnail visual grammar.
- exports are labeled by purpose and platform,
- the package is usable without extra manual work.

## Common Pitfalls

- Writing generic metadata that lacks search relevance.
- Failing to feature or contextualize the thumbnail persona/claim within the first minutes of the script.
- Mixing platform variants without clear labels.

---

## Gate Reminder (Binding)

This stage gates on human approval (`human_approval_default: true`). After review passes:
checkpoint with `status="awaiting_human"`, present the summary (the Backlot board renders
the artifact), and **END YOUR TURN**. Do not start the next stage in the same response.
Approval is per-gate — an earlier "go ahead" does not cover this gate.
