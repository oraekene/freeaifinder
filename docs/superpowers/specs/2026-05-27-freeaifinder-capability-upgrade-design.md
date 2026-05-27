# FreeAI Finder — Capability Upgrade Design

## Overview

Upgrade the FreeAI Finder single-page HTML application from a flat model-ranking site to a
multi-capability model comparison tool. Users can compare models across Text, Code, Math,
Image, Vision, Video, Audio, and Embedding capabilities, each with per-capability benchmarks,
tiered pricing, and user-selectable composite scoring.

## Phased Approach

The work is split into 4 independent phases, each delivering a usable result:

1. **Data Audit** — Research and correct all existing model/provider data
2. **Capability Data Model** — Restructure from flat arrays to capability-based objects
3. **Capability UI** — New expansion panels, tier badges, per-capability filtering
4. **Advanced Scoring** — User-selectable benchmark ranking, BCWES/MCWES composites

---

## Phase 1: Data Audit

### Scope
- ~500 model entries in `MODELS` array
- ~120 provider entries in `PROVIDERS` array
- ~40 pricing profiles in `PP` object

### Method
1. **Free tier providers first** — Cerebras, Groq, Cloudflare, Google, OpenRouter, GitHub
   Models, Cohere, Mistral, SiliconFlow, Z.AI — verify model names, pricing, rate limits
2. **Major creators** — OpenAI, Google, Meta, DeepSeek, Mistral, Alibaba, xAI — verify
   latest models and benchmarks
3. **Spot-check benchmarks** — MMLU Pro, GPQA, AIME, LiveCodeBench against published sources
4. **Capability benchmark research** — For image, vision, video, audio, embeddings:
   identify relevant public benchmarks (e.g. MMMU, MathVista for vision; MTEB for embeddings)
5. **Stale entries** — Flag providers/creators with no verified public info as `unknown`

### Output
Corrected `MODELS`, `PROVIDERS`, `PP` data ready for Phase 2 transformation.

---

## Phase 2: Capability Data Model

### Data Structure

Each model moves from a flat array to a named object:

```js
{
  name: "GPT-5.5 (high)",
  slug: "gpt-5.5-high",
  creator: "OpenAI",
  releaseDate: "2026-04",
  capabilities: {
    "text": {
      benchmarks: {
        "artificial-analysis": 58.9,
        "mmlu-pro": 93.2,
        "gpqa": 43.0,
        "hle": null,
        "ifbench": null
      },
      tiers: {
        "free": null,
        "paid": [
          { name: "Standard", inputPrice: 5.0, outputPrice: 30.0 }
        ]
      }
    },
    "code": {
      benchmarks: {
        "artificial-analysis": 58.5,
        "livecodebench": null,
        "swe-bench": null
      },
      tiers: { ... }
    }
  }
}
```

### Capability Registry

| ID | Label | Default benchmark | Available benchmarks |
|----|-------|-------------------|---------------------|
| text | Text / Chat | artificial-analysis | artificial-analysis, mmlu-pro, gpqa, hle, ifbench |
| code | Code | artificial-analysis | artificial-analysis, livecodebench, swe-bench |
| math | Math | artificial-analysis | artificial-analysis, aime-2025, math-500 |
| image | Image Gen | none | none (quality benchmarks TBD) |
| vision | Vision | none | mmmu, mathvista |
| video | Video | none | none (TBD) |
| audio | Audio / Speech | none | none (TBD) |
| embeddings | Embeddings | none | mteb |

### Provider Data

`PROVIDERS` + `PP` merged into richer provider objects:
```js
{
  name: "Groq",
  cat: "free",
  badge: "Permanent free",
  url: "https://console.groq.com",
  freeDescription: "30 RPM · 14,400 RPD · No CC required",
  speedNote: "LPU: 300-1000 t/s",
  creators: ["Meta", "Alibaba", "OpenAI", "Kimi", "Mistral"],
  models: {
    "gpt-oss-120b": {
      text: { inputPrice: 0.15, outputPrice: 0.60, speed: 500, ttft: 0.59 },
      code: { inputPrice: 0.15, outputPrice: 0.60 },
      vision: null
    }
  }
}
```

### Migration
- Write `transformLegacyModel(arr)` and `transformLegacyProvider(obj)` converters
- All render functions updated to access `m.capabilities.text.benchmarks["mmlu-pro"]`
  instead of `m[8]`
- Old `PP` pricing merged into model objects under their capability `tiers`.
  Mapping: `PP[provider].models[modelKey]` contains `{i, o, spd, ttft}`. This
  becomes the `paid` tier (first paid tier) for the `text` capability. If both
  `i` and `o` are `0`, it becomes the `free` tier instead.
- All in-place in `index.html` (no file splitting)

---

## Phase 3: Capability UI

### Provider Panel Redesign

Each provider's expansion row gains capability sub-tabs:

```
Provider: Groq
├── [Text] [Code] [Vision] [Image] [Audio]   ← capability tabs
│
├── Text mode:
│   ├── Model | Benchmark | Free Tier | Paid Tiers
│   ├── llama-3.3-70b | 58.9 | ✅ Free | $0.59/$0.79
│   ├── gpt-oss-120b | 33.3 | ✅ Free | $0.15/$0.60
│   └── ...
│
├── Code mode:
│   ├── Model | Benchmark | Free Tier | Paid Tiers
│   └── ...
```

### Components

1. **Capability tab bar** — buttons inside each expandable provider row; switching changes
   the model table below to show only models with that capability
2. **Per-capability model table** — columns: model name, benchmark score (with bar),
   free tier indicator + rate limits, paid tier prices (all tiers listed)
3. **Badge system** — each model row shows its tiers as badges:
   - `Free` badge (green) — has a free tier for this capability
   - `Paid` badges per tier — Standard, Pro, Enterprise with prices
   - Hover/click on free badge expands rate limit details
4. **Global filters** — persist across all tabs:
   - Capability selector dropdown (which capability to show across panels)
   - "Free only" checkbox — filters to models with a free tier for selected capability
   - Toggle to show paid tiers
5. **Column sorting** — sort by benchmark score, price, speed within each capability view

### Models Tab (Tab 2)

Currently sorted by intelligence. Updated to sort by the **selected capability's default
benchmark**. Users can switch the ranking capability from a dropdown.

### Cross-Reference Tab (Tab 3)

Both "By Model" and "By Provider" views gain the capability selector. The provider
ranking uses the selected capability's benchmark for BWECS, Marginal BWECS,
and the new BCWES/MCWES scores.

---

## Phase 4: Advanced Scoring

### User-Selectable Benchmark

A dropdown in the Xref tab controls lets users pick:
1. **Capability**: text, code, math, vision, etc.
2. **Benchmark**: within that capability, which benchmark is the primary ranking signal

When changed, all composite scores recalculate.

### Composite Scores

**BCWES** (Budget Capability-Weighted Elite Score):
- Same formula as current BWECS but uses the user-selected capability benchmark
  instead of hardcoded intelligence
- Combines: band score (S/A/B/C) × price weight + secondary signal boost

**MCWES** (Marginal Capability-Weighted Elite Score):
- Marginal (exclusive-value) version of BCWES
- Each provider scored only on models not already counted by higher-ranked providers

**Band breakdown** ("2-18-22-5" style score):
- Shows S/A/B/C counts as a compact string (e.g. "S:2 A:18 B:22 C:5")
- Visible in the provider summary columns

### UI Changes

- Xref "By Provider" controls gain capability + benchmark selectors
- BWECS column relabeled to BCWES when a non-default benchmark is selected
- Marginal column relabeled to MCWES
- Band breakdown column added to table

---

## Implementation Order

1. Phase 1 (Data Audit) — independent, can start immediately
2. Phase 2 (Data Model) — requires Phase 1 data as input
3. Phase 3 (Capability UI) — requires Phase 2 data model
4. Phase 4 (Advanced Scoring) — requires Phase 3 UI

Each phase is self-contained and testable before the next begins.

## Key Constraints

- Single HTML file must be preserved (no external data files)
- Dark mode must continue working
- Existing advanced boolean search must continue to function
- All existing tabs (Providers, Models, XRef, Chart, FAQ) must continue working
