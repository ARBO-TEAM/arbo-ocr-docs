---
title: Accuracy defaults
---

# Accuracy defaults (read this if you upgrade)

!!! warning "Defaults changed in a recent accuracy cycle — impact is large"

    On full-page text quality, the SROIE receipt smoke went from **~84% → ~94%**
    char similarity for cold arbo `small` — within ~0.2 pts of a TS PaddleOCR
    port on the same images.

    If you pin config or re-implement the pipeline yourself, note **every** item
    below. Three of them change behaviour your code may already depend on.

## What changed

| Setting | Old | New default | Why it matters |
|---------|-----|-------------|----------------|
| `modelType` | `medium` | **`small`** | Medium is ~4× slower on CPU with little sim gain on receipts. Use medium only after measuring on *your* data (GPU helps). |
| `detLimitSideLen` | `1536` | **`960`** | Longest side for det resize. 1536 over-merged / hurt full-page score on dense receipts; 960 recovered ~2 pts. Override for very high-res pages if boxes look wrong. |
| Reading order | det order only | **always sorted** (centroid y then x) | Full-page string metrics and human reading depend on order. `page.lines` is now top→bottom, left→right — **not** raw detector order. |
| CTC post | greedy only | **gap→space + fullwidth→ASCII** | Spaces injected where CTC timesteps show wide gaps (columnar fields). Fullwidth punctuation (e.g. `：`) maps to ASCII on non-CJK text. Always on in the recognizer. |
| `minimumConfidence` | (none) | **`0.5`** | Drops low-confidence lines (Paddle `drop_score`). Pure symbol lines need **0.8**. Set `0` to keep every box (legacy). |
| `splitOvermerged` | n/a | **`false`** | Optional ink-gap split of wide fused det boxes. **Off** by default — aggressive split *lost* ~1 pt on the smoke set. Enable only when det clearly fuses side-by-side fields. |
| `LinePrediction.score` | det-ish / unclear | **rec mean CTC conf** | New `detScore` holds the detector box score. JSON / Python: `score` + `det_score`. |
| CLI `--model-type` | medium | **small** | Matches library default. |

## Measured

Warm Python `Engine`, CPU, 5 SROIE receipts, char sim vs box GT:

| Engine | Size | Avg sim | Avg ms |
|--------|------|--------:|-------:|
| arbo | tiny | 93.3% | ~165 |
| arbo | **small** | **94.4%** | ~530 |
| arbo | medium | 94.8% | ~2120 |
| ppu-paddle-ocr (ref) | small | 94.6% | ~793 |

The reference row is the point of the table: at `small`, arbo lands within
~0.2 pts of the PaddleOCR port while running faster on the same images.
`medium` is ahead on similarity and roughly 4× the latency of `small`.

## What to watch when integrating

!!! danger "Items 1, 2 and 6 are breaking for existing integrations"

    **1. Line order** — indexes into `page.lines` no longer mean what they used
    to. **2. `minimumConfidence = 0.5`** — lines you used to receive are now
    silently dropped. **6. `score` semantics** — the number changed meaning, so
    thresholds built on it are now wrong without any error being raised. Each of
    these can pass your build and fail your data.

1. :material-alert-octagon: **Breaking — line order changed.** Anything that
   assumed detector emission order (or matched boxes by index against an old
   dump) must use polygon geometry or re-sort the same way.
2. :material-alert-octagon: **Breaking — `minimumConfidence = 0.5` drops
   lines.** Low-contrast logos, rules read as `+-`, and weak boxes disappear
   from `page.lines`. For audit dumps that need every box, set
   `minimumConfidence = 0`.
3. **Do not "fix accuracy" by switching to medium first.** Diagnosis on the
   smoke set: matched-line rec was already ~97%; the gap was layout / order /
   det scale, not rec capacity. Medium buys ~+0.4 pts for ~4× latency on CPU.
4. **`detLimitSideLen` is not "bigger = better".** Try 960 before 1280/1536 on
   receipts; re-measure if your docs are posters or A3 scans.
5. **Det file is still one size.** `modelType` only selects the recognizer.
   Pairing `det_small` / `det_medium` ONNX via `detModelPath` did not help on
   the smoke set; keep the default `*_det.onnx` unless you A/B otherwise. See
   [Choosing a model size](sizes.md).
6. :material-alert-octagon: **Breaking for score consumers.** If you treated
   `score` as det confidence, switch to `detScore` / `det_score`.
7. **Opt-in only:** `useClahe`, `useAngleCls`, `splitOvermerged` — leave off
   unless the failure mode matches (faded scans / 180° / confirmed over-merge).
   See [Low-contrast documents (CLAHE)](clahe.md).

## Typical production CPU defaults

```cpp
// Typical production CPU defaults after the accuracy cycle (these are already
// the EngineConfig defaults — shown explicitly for upgrades):
arbo::ocr::EngineConfig cfg;
cfg.modelsDir = "models";
cfg.modelType = "small";
cfg.detLimitSideLen = 960;
cfg.minimumConfidence = 0.5f;  // 0 = keep all boxes
// cfg.splitOvermerged = true; // only if side-by-side fields fuse
// cfg.useClahe = true;        // only if det misses low-contrast text
```

!!! note "You do not need to set any of these"

    Every assignment above is already the default. It is written out so an
    upgrading integration can diff its own pinned config against the new
    baseline line by line.

Related: [Choosing a model size](sizes.md) ·
[Low-contrast documents (CLAHE)](clahe.md) ·
[API Reference](../api/index.md) · [Benchmarks](../benchmarks.md)
