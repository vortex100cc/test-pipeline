---
name: ai-text-detector
description: Score and evaluate English text for AI-generation likelihood using local DeBERTa-v3-large on GPU/CPU.
---

# AI Text Detector

Local binary sequence classifier based on `ShantanuT01/gradient-ai-text-detector` (`microsoft/deberta-v3-large`).
Runs locally on NVIDIA GPU (`cuda:0` / GTX 1070) with automatic CPU fallback.

## When to Use
- **Script Review & Pass 2 Humanization**: Check candidate narration sections or entire scripts before finalizing.
- **A/B Variation Testing**: Compare raw LLM drafts against humanized/rephrased versions to confirm slop reduction.
- **Objective Quality Gating**: Reject script beats with $P(AI) > 0.50$ (or target threshold).

## Context Length & Sliding Window (Texts > 512 Tokens)
- **Architecture Limit:** The underlying DeBERTa-v3-large model has a fixed positional embedding table of 512 tokens (~350–400 English words).
- **Short Texts (<= 510 tokens):** Evaluated in a single forward pass.
- **Long Texts / Entire Scripts (> 510 tokens):** Automatically chunked using a sliding window (510 tokens with 256-token overlap / 50% stride). 100% of the script is evaluated, producing:
  - `p_ai`: Mean probability across all chunks.
  - `p_ai_max`: Worst-case chunk probability (flags if an AI-slop paragraph is hidden in a long text).
  - `chunks`: List of per-chunk scores and token lengths.

## Mathematical Foundation: Why Sigmoid and Logits
According to official model documentation (`ShantanuT01/gradient-ai-text-detector`):
> *Architecture: DeBERTa-v3-large with a single classification head (binary, sigmoid output).*
> *Output: A single scalar P(AI) in [0, 1]; a decision threshold of 0.5 is used by default, where P(AI) > 0.5 indicates AI-generated text.*

The model's linear head outputs a raw **logit** $z \in (-\infty, +\infty)$ (log-odds $\ln \frac{p}{1-p}$).
Standard logistic sigmoid converts log-odds into true probability:
$$P(AI) = \sigma(z) = \frac{1}{1 + e^{-z}}$$

- **No arbitrary bounds needed:** Sigmoid is mathematically defined on $(-\infty, +\infty) \to (0, 1)$.
- **Logit = 0.0:** Exact 50/50 balance ($P(AI) = 50\%$).
- **Logit = -1.13:** $P(AI) = 24.4\%$ (75.6% Human).
- **Logit = +2.20:** $P(AI) = 90.0\%$ (10.0% Human).
- **Logit = +7.60:** $P(AI) = 99.95\%$ (Pure AI Slop).

## Python API (Agent / Pipeline Usage)

No shell or escaping needed; pass text directly in memory:

```python
from tools.analysis.ai_text_detector import AiTextDetector

detector = AiTextDetector()
result = detector.execute({
    "text": "Your script narration text here...",
    "threshold": 0.5,      # Optional, default: 0.5
})

if result.success:
    p_ai = result.data["p_ai"]                  # Probability AI [0.0..1.0]
    p_human = result.data["p_human"]            # Probability Human [0.0..1.0] (1.0 - p_ai)
    p_ai_max = result.data["p_ai_max"]          # Peak AI probability across chunks
    is_ai = result.data["is_ai"]                # True if p_ai >= threshold
    label = result.data["confidence_label"]     # "AI_GENERATED" or "HUMAN_WRITTEN"
    logit = result.data["logit"]                # Raw logit score
    chunks = result.data["chunks"]              # Per-chunk breakdown if text > 512 tokens
```

## CLI Usage (Manual Terminal Verification)

From repository root:

```powershell
# 1. Built-in test suite (runs on 3 standard samples)
uv run python tools/analysis/ai_text_detector.py

# 2. Score a custom string
uv run python tools/analysis/ai_text_detector.py "Text to check"

# 3. Score an entire script file (no PowerShell quote escaping issues)
uv run python tools/analysis/ai_text_detector.py --file path/to/script.txt
```

## Output Metrics Interpretation

| Metric | Range | Meaning |
| :--- | :--- | :--- |
| **`p_ai`** | `0.0 .. 1.0` | Probability that text is AI-generated ($P(AI) = \sigma(\text{logit})$). |
| **`p_human`** | `0.0 .. 1.0` | Probability that text is Human-written ($1.0 - P(AI)$). |
| **`threshold`** | `0.5` (default) | Decision boundary: $P(AI) \ge \text{threshold} \implies$ `AI_GENERATED`. |
| **`logit`** | `(-inf .. +inf)` | Raw linear confidence score before sigmoid. |
| **`total_tokens`** | integer | Wordpiece token count. If $> 510$, multi-chunk windowing was applied. |

## Target Guidelines for Humanization (Pass 2)
- **AI Slop**: typically scores $P(AI) \ge 0.90$ ($\text{logit} \ge +2.2$).
- **Neutral / Informational**: typically scores $P(AI) \in [0.15 .. 0.40]$.
- **Target for Pass 2 (Humanized)**: strive for $P(AI) \le 0.35$ while preserving 100% of facts and figures.
