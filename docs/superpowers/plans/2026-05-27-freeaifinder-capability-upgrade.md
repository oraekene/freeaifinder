# FreeAI Finder — Capability Upgrade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Upgrade FreeAI Finder from a flat model-ranking site to a multi-capability model comparison tool with per-capability benchmarks, tiered pricing, and user-selectable composite scoring.

**Architecture:** Single HTML file with embedded JS data. All changes go into `index.html`. A `capabilities` registry defines available capability types (text, code, math, image, vision, video, audio, embeddings). Model data restructured from flat arrays to objects with per-capability benchmark and tier sub-objects. Render functions updated to use named property access.

**Tech Stack:** Vanilla JS, Chart.js (CDN), no build tools, single HTML file.

---

## Phase 1: Data Audit

Research and correct all existing model/provider data. No code changes.

### Task 1a: Research Free Tier Providers

**Files:** None (research output feeds into index.html data)

- [ ] **Research Cerebras, Groq, Cloudflare Workers AI, Google**: Verify model names, pricing, rate limits, free tier details for each. Check their latest published docs.
- [ ] **Research OpenRouter, GitHub Models, Cohere, Mistral**: Same — verify all model listings, free tier limits, pricing for paid tiers.
- [ ] **Research SiliconFlow, Z.AI, Zhipu AI, ModelScope**: Verify free tier models, rate limits, pricing.
- [ ] **Research Ollama Cloud, Venice AI, Kilo Gateway, Hugging Face**: Verify free tier details.

### Task 1b: Research Major Creators

- [ ] **Research OpenAI**: Verify GPT-5.5, o3, GPT-5.4 series model names, pricing, benchmark scores against published data.
- [ ] **Research Google**: Verify Gemini 2.5 Pro/Flash/Flash-Lite model names, pricing, benchmarks.
- [ ] **Research Meta**: Verify Llama 4, Llama 3.3, Llama 3.1 model names and benchmarks.
- [ ] **Research DeepSeek, Mistral, Alibaba, xAI**: Verify latest model names, pricing, benchmarks.
- [ ] **Spot-check key benchmarks**: Verify MMLU Pro, GPQA, AIME, LiveCodeBench scores for OpenAI, Google, Meta, DeepSeek models against published leaderboards.

### Task 1c: Research Capability Benchmarks

- [ ] **Research vision benchmarks**: Identify MMMU, MathVista, and other standard vision benchmarks. Get recent scores for major vision-capable models.
- [ ] **Research embedding benchmarks**: Identify MTEB and other embedding evaluation standards.
- [ ] **Research image/video/audio benchmarks**: Identify available benchmarks for these capabilities, even if not yet widely adopted.

### Task 1d: Catalog Corrections

- [ ] **Compile all corrections**: Consolidate research findings into a structured correction list: added/removed models, updated prices, corrected benchmark scores, changed rate limits.
- [ ] **Preview impact**: Count how many entries change — flag high-impact corrections for user review.

---

## Phase 2: Capability Data Model Restructure

Transform `index.html` from flat array data model to capability-based object model.

### Task 2a: Add Capability Registry

**Files:**
- Modify: `index.html` (add to JS section, before MODELS definition)

- [ ] **Define the capabilities registry object**:

In the JS section (around line 1305, before `const MODELS`), add:

```js
const CAPABILITIES = {
  text: {
    label: "Text / Chat",
    defaultBenchmark: "artificial-analysis",
    benchmarks: ["artificial-analysis", "mmlu-pro", "gpqa", "hle", "ifbench"],
    benchmarkLabels: { "artificial-analysis": "Artificial Analysis", "mmlu-pro": "MMLU Pro", "gpqa": "GPQA", "hle": "HLE", "ifbench": "IFBench" }
  },
  code: {
    label: "Code",
    defaultBenchmark: "artificial-analysis",
    benchmarks: ["artificial-analysis", "livecodebench", "swe-bench"],
    benchmarkLabels: { "artificial-analysis": "Artificial Analysis", "livecodebench": "LiveCodeBench", "swe-bench": "SWE-bench" }
  },
  math: {
    label: "Math",
    defaultBenchmark: "artificial-analysis",
    benchmarks: ["artificial-analysis", "aime-2025", "math-500"],
    benchmarkLabels: { "artificial-analysis": "Artificial Analysis", "aime-2025": "AIME 2025", "math-500": "MATH-500" }
  },
  image: {
    label: "Image Gen",
    defaultBenchmark: null,
    benchmarks: [],
    benchmarkLabels: {}
  },
  vision: {
    label: "Vision",
    defaultBenchmark: null,
    benchmarks: ["mmmu", "mathvista"],
    benchmarkLabels: { "mmmu": "MMMU", "mathvista": "MathVista" }
  },
  video: {
    label: "Video",
    defaultBenchmark: null,
    benchmarks: [],
    benchmarkLabels: {}
  },
  audio: {
    label: "Audio / Speech",
    defaultBenchmark: null,
    benchmarks: [],
    benchmarkLabels: {}
  },
  embeddings: {
    label: "Embeddings",
    defaultBenchmark: null,
    benchmarks: ["mteb"],
    benchmarkLabels: { "mteb": "MTEB" }
  }
};
```

- [ ] **Verify the registry loads correctly**: Open `index.html` in browser, check console that `CAPABILITIES` is defined.

### Task 2b: Write Data Transform Functions

**Files:**
- Modify: `index.html` (add transform functions after CAPABILITIES, before MODELS)

- [ ] **Add `transformLegacyModel()` function**:

```js
function transformLegacyModel(arr) {
  const [
    name, creator, date,
    intel, code, math, speed, ttft,
    mmlu, gpqa, hle, lcb, math500, aime, ifbench, _blank,
    price, input, output
  ] = arr;

  const cap = {};

  // Text capability
  cap.text = {
    benchmarks: {
      "artificial-analysis": intel ?? null,
      "mmlu-pro": mmlu ?? null,
      "gpqa": gpqa ?? null,
      "hle": hle ?? null,
      "ifbench": ifbench ?? null
    },
    tiers: {}
  };
  if (price === 0) {
    cap.text.tiers.free = { inputPrice: 0, outputPrice: 0, rateLimit: null };
  } else if (price != null) {
    cap.text.tiers.paid = [{ name: "Standard", inputPrice: input ?? price, outputPrice: output ?? price }];
  }

  // Code capability (inherit tiers from text for now)
  cap.code = {
    benchmarks: {
      "artificial-analysis": code ?? null,
      "livecodebench": lcb ?? null,
      "swe-bench": null
    },
    tiers: JSON.parse(JSON.stringify(cap.text.tiers))
  };

  // Math capability
  cap.math = {
    benchmarks: {
      "artificial-analysis": math ?? null,
      "aime-2025": aime ?? null,
      "math-500": math500 ?? null
    },
    tiers: JSON.parse(JSON.stringify(cap.text.tiers))
  };

  // Empty capabilities (filled in Phase 1 data cleanup)
  cap.image = { benchmarks: {}, tiers: {} };
  cap.vision = { benchmarks: { "mmmu": null, "mathvista": null }, tiers: JSON.parse(JSON.stringify(cap.text.tiers)) };
  cap.video = { benchmarks: {}, tiers: {} };
  cap.audio = { benchmarks: {}, tiers: {} };
  cap.embeddings = { benchmarks: { "mteb": null }, tiers: JSON.parse(JSON.stringify(cap.text.tiers)) };

  return {
    name,
    slug: name.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/(^-|-$)/g, ''),
    creator,
    releaseDate: date,
    capabilities: cap,
    // Legacy accessors for backward compatibility during migration
    _legacy: arr
  };
}
```

- [ ] **Add `transformAllData()` function**:

```js
let TRANSFORMED = false;
function transformAllData() {
  if (TRANSFORMED) return;
  window.__MODELS_OBJ = MODELS.map(transformLegacyModel);
  TRANSFORMED = true;
}
```

- [ ] **Verify transforms work**: Call `transformAllData()` in console, inspect output.

### Task 2c: Add Per-Provider Pricing Injection

- [ ] **Write function to merge PP pricing into capability tiers**:

```js
function injectProviderPrices(providerName, modelObj) {
  const pp = PP[providerName];
  if (!pp) return;
  const norm = modelObj.slug;
  const pm = Object.entries(pp.models).find(([k]) => norm.includes(k) || k.includes(norm.slice(0, 8)));
  if (!pm) return;
  const [, prices] = pm;
  Object.values(modelObj.capabilities).forEach(cap => {
    if (prices.i === 0 && prices.o === 0) {
      cap.tiers.free = { inputPrice: 0, outputPrice: 0, rateLimit: pp.free || null };
    } else if (prices.i != null || prices.o != null) {
      if (!cap.tiers.paid || !cap.tiers.paid.length) {
        cap.tiers.paid = [{ name: "Standard", inputPrice: prices.i, outputPrice: prices.o }];
      }
    }
    if (prices.spd != null && !cap.speed) cap.speed = prices.spd;
    if (prices.ttft != null && !cap.ttft) cap.ttft = prices.ttft;
  });
}
```

- [ ] **Wire into transform**: Call `injectProviderPrices` inside `transformLegacyModel` or in a post-processing step.

### Task 2d: Add Global Capability State

- [ ] **Add capability state variables**:

```js
let activeCapability = 'text';
let activeBenchmark = 'artificial-analysis';
let freeOnlyCapability = false;
```

- [ ] **Add capability getter helper**:

```js
function getCapBenchmark(model, capId, benchId) {
  const cap = model.capabilities[capId];
  if (!cap) return null;
  if (benchId) return cap.benchmarks[benchId] ?? null;
  return cap.benchmarks[capabilities[capId]?.defaultBenchmark] ?? null;
}

function getCapTierSummary(model, capId) {
  const cap = model.capabilities[capId];
  if (!cap) return { hasFree: false, hasPaid: false, tiers: [] };
  const tiers = [];
  if (cap.tiers.free) tiers.push({ type: 'free', ...cap.tiers.free });
  if (cap.tiers.paid) cap.tiers.paid.forEach((t, i) => tiers.push({ type: 'paid', index: i, ...t }));
  return {
    hasFree: !!cap.tiers.free,
    hasPaid: !!cap.tiers.paid?.length,
    tiers
  };
}
```

- [ ] **Verify in browser**: Load page, call `getCapBenchmark(window.__MODELS_OBJ[0], 'text', 'mmlu-pro')` and verify it returns a number.

---

## Phase 3: Capability UI

### Task 3a: Add Capability Selector to Controls

**Files:**
- Modify: `index.html` (controls bar in each tab, CSS for selector)

- [ ] **Add capability selector dropdown HTML** to the controls bar in the Providers tab (after the existing search):

```html
<select id="p1-capability" style="max-width:150px">
  <option value="text">Text / Chat</option>
  <option value="code">Code</option>
  <option value="math">Math</option>
  <option value="image">Image Gen</option>
  <option value="vision">Vision</option>
  <option value="video">Video</option>
  <option value="audio">Audio / Speech</option>
  <option value="embeddings">Embeddings</option>
</select>
```

- [ ] **Add same selector to Models tab** (Tab 2) and **XRef tab** (Tab 3) controls.
- [ ] **Add "Free only" checkbox** to each tab's controls:

```html
<label class="tog"><input type="checkbox" id="p1-free-cap"> Free only</label>
```

- [ ] **Wire event listeners**:

```js
document.getElementById('p1-capability').addEventListener('change', e => {
  activeCapability = e.target.value;
  rerenderAll();
});
document.getElementById('p1-free-cap').addEventListener('change', e => {
  freeOnlyCapability = e.target.checked;
  rerenderAll();
});
```

### Task 3b: Redesign Provider Panel for Capabilities

- [ ] **Modify `renderP1()`** to show capability tabs inside each provider's accordion:

When a provider row expands, instead of showing a flat model table, show:

```html
<div class="cap-tabs">
  <button class="cap-tab active" data-cap="text">Text / Chat</button>
  <button class="cap-tab" data-cap="code">Code</button>
  <button class="cap-tab" data-cap="vision">Vision</button>
  <button class="cap-tab" data-cap="image">Image Gen</button>
</div>
<div class="cap-panel" data-cap="text">
  <!-- Model table filtered to text-capable models -->
</div>
```

- [ ] **Add CSS for capability tabs**:

```css
.cap-tabs{display:flex;gap:4px;margin-bottom:10px;flex-wrap:wrap}
.cap-tab{padding:6px 12px;border-radius:999px;border:1px solid var(--bdr2);background:transparent;color:var(--muted);font-size:.75rem;cursor:pointer;letter-spacing:.06em;text-transform:uppercase}
.cap-tab.active{background:var(--accent);color:#fff;border-color:var(--accent)}
```

- [ ] **Modify `renderP1InnerMods()`** to accept a `capabilityId` parameter and filter/sort models by that capability's benchmark:

```js
function renderP1InnerMods(p, sortState, capabilityId) {
  const mods = getModsForProv(p);
  const capId = capabilityId || activeCapability;
  const benchId = activeBenchmark;
  // Filter to only models that have this capability with data
  const capMods = mods.filter(m => m.capabilities[capId]?.benchmarks);
  // Sort by the capability's benchmark
  const col = sortState.col, d = sortState.dir;
  capMods.sort((a, b) => {
    const sa = getCapBenchmark(a, capId, benchId) ?? -1;
    const sb = getCapBenchmark(b, capId, benchId) ?? -1;
    return d * (sb - sa);
  });
  // Render table: Model | Creator | Benchmark Score | Speed | Free/Paid Tiers
  const rows = capMods.map(m => {
    const score = getCapBenchmark(m, capId, benchId);
    const summary = getCapTierSummary(m, capId);
    return `<tr>
      <td>${m.name}</td>
      <td class="dim">${m.creator}</td>
      <td style="text-align:right">${score != null ? score.toFixed(1) : '<span class="dim">—</span>'}</td>
      <td style="text-align:right">${m.capabilities[capId].speed ? Math.round(m.capabilities[capId].speed) + ' t/s' : '<span class="dim">—</span>'}</td>
      <td>${summary.hasFree ? '<span class="free-pill">Free</span>' : ''} ${summary.hasPaid ? '$' + summary.tiers.filter(t => t.type === 'paid').map(t => t.inputPrice?.toFixed(2)).join('/') : ''}</td>
    </tr>`;
  }).join('');
  return `<table class="inner-tbl">...</table>`;
}
```

- [ ] **Add tier display** in the model table:

For each model row, show a compact tier summary:
```html
<td>
  <span class="free-pill">Free: 30 RPM / 14K RPD</span>
  <details><summary>Paid: $0.59/$0.79</summary>...full details...</details>
</td>
```

### Task 3c: Add Badge System

- [ ] **Add badge rendering function**:

```js
function renderCapabilityBadges(model, capId) {
  const cap = model.capabilities[capId];
  if (!cap) return '';
  const parts = [];
  if (cap.tiers.free) {
    const rl = cap.tiers.free.rateLimit ? ` — ${cap.tiers.free.rateLimit}` : '';
    parts.push(`<span class="badge bg" title="Free tier${rl}">Free</span>`);
  }
  if (cap.tiers.paid) {
    cap.tiers.paid.forEach((tier, i) => {
      const price = tier.inputPrice != null ? `$${tier.inputPrice.toFixed(2)}/$${tier.outputPrice.toFixed(2)}` : '';
      parts.push(`<span class="badge bb" title="${tier.name}: ${price}">${tier.name}</span>`);
    });
  }
  return parts.join(' ');
}
```

- [ ] **Add filtering by badge**: Users can click a badge to filter the table to only models with that tier.

### Task 3d: Add Benchmark Display

- [ ] **Add benchmark display per capability**:

Inside each capability panel, show the benchmark scores for the relevant benchmarks:

```html
<div class="bench-grid">
  <div class="bench-item"><div class="bench-lbl">Artificial Analysis</div><div class="bench-val">58.9</div></div>
  <div class="bench-item"><div class="bench-lbl">MMLU Pro</div><div class="bench-val">93.2</div></div>
</div>
```

- [ ] **Wire the benchmark selector** to allow changing which benchmark drives sorting in each capability.

### Task 3e: Update Models Tab (Tab 2)

- [ ] **Update `getM2()`** to use the active capability's default benchmark for the intelligence column:

```js
function getM2Score(m) {
  return getCapBenchmark(m, activeCapability, activeBenchmark);
}
```

- [ ] **Relabel the "Intelligence" column header** to show the active capability name, e.g. "Text AI" or "Code AI".

### Task 3f: Update Cross-Reference Tab (Tab 3)

- [ ] **Add capability selector** to both "By Model" and "By Provider" modes.
- [ ] **Update `getBestForProv()`** to use the active capability's benchmark.
- [ ] **Update `getXMList()`** to sort by the active capability's benchmark.

---

## Phase 4: Advanced Scoring

### Task 4a: Benchmark Selector UI

- [ ] **Add benchmark dropdown** to Xref "By Provider" controls:

```html
<select id="xp-benchmark">
  <!-- dynamically populated based on selected capability -->
</select>
```

- [ ] **Dynamic benchmark options**: When capability changes, update the benchmark dropdown to show only benchmarks relevant to that capability (from `CAPABILITIES[capId].benchmarks`).

- [ ] **Persist selection**: `activeBenchmark` state variable updated on change.

### Task 4b: Update BWECS to Use Selected Benchmark

- [ ] **Modify `getRankingSignals()`** to accept a `benchmarkId` parameter:

```js
function getRankingSignals(m, rankingMode = getSelectedPrimaryMode(), domainKey = '', benchmarkId = activeBenchmark) {
  const signal = getCapBenchmark(m, activeCapability, benchmarkId);
  const ttft = m.capabilities[activeCapability]?.ttft ?? m.capabilities.text?.ttft ?? null;
  const speed = m.capabilities[activeCapability]?.speed ?? m.capabilities.text?.speed ?? null;
  // Use signal as the intelligence score, ttft/speed from the capability
  const intelPrimary = Number.isFinite(signal) ? signal : (Number.isFinite(ttft) ? -ttft : -Infinity);
  const latencyPrimary = Number.isFinite(ttft) ? -ttft : -Infinity;
  const speedPrimary = Number.isFinite(speed) ? speed : -Infinity;
  const primary = rankingMode === 'speed' ? speedPrimary : rankingMode === 'latency' ? latencyPrimary : intelPrimary;
  const secondary = rankingMode === 'speed' ? (Number.isFinite(signal) ? signal : -Infinity) : (Number.isFinite(ttft) ? -ttft : -Infinity);
  const display = rankingMode === 'speed' ? (Number.isFinite(speed) ? speed.toFixed(0) + ' t/s' : '—') : rankingMode === 'latency' ? (Number.isFinite(ttft) ? ttft.toFixed(1) + 's' : '—') : (Number.isFinite(signal) ? signal.toFixed(1) : '—');
  return { signal, ttft, speed, primary, secondary, display, source: rankingMode };
}
```

- [ ] **Update `getBudgetWeightedEliteStats()`** to pass benchmarkId through.
- [ ] **Update `buildMarginalBwecsRanking()`** to pass benchmarkId through.

### Task 4c: Add BCWES/MCWES Display

- [ ] **Add BCWES column** (relabeled BWECS when non-default benchmark is selected):

In the Xref "By Provider" column headers:
```html
<span class="ch srt" data-xpcol="bwecs" style="text-align:right">BWECS/BCWES</span>
```

- [ ] **Add MCWES column**:

```html
<span class="ch srt" data-xpcol="xbwecs" style="text-align:right">Marginal BWECS/MCWES</span>
```

- [ ] **Add band breakdown column** ("2-18-22-5" style):

```html
<span class="ch srt" data-xpcol="bands" style="text-align:right">Bands S·A·B·C</span>
```

- [ ] **Render band breakdown** in provider rows:

```js
function renderBandBreakdown(stats) {
  return `<span style="font-size:11px">S:${stats.s} A:${stats.a} B:${stats.b} C:${stats.c}</span>`;
}
```

### Task 4d: Update Scoring Documentation

- [ ] **Update the budget note in `renderXP()`** to reference BCWES/MCWES when non-default benchmarks are used.
- [ ] **Ensure the ledger/legend text** explains what BCWES and MCWES mean.

---

## Self-Review Checklist

After writing the plan, verify:
- [ ] Every step in Phase 2-4 has executable code (not "TBD" or "implement later")
- [ ] All function names used in later tasks match their definitions in earlier tasks
- [ ] Every task references the exact file being modified (`index.html`)
- [ ] The plan covers all 4 phases from the spec
- [ ] Phase 1 tasks have clear research targets, not vague "research everything"
