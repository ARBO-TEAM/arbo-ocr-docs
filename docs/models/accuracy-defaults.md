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

    Two further behaviour changes landed *after* that cycle — detector input
    scaling and reading-order tolerance. See
    [Changes since the accuracy cycle](#changes-since-the-accuracy-cycle).

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

Warm `Engine`, CPU, the same 5 SROIE receipts for every row, char sim vs box
GT. Latency is **engine** time — the OCR call only, excluding process start
and language-side marshalling — because that is the one number all three
engines report comparably. Reference engines are `ppu-paddle-ocr` and
`rapidocr` **3.9.2**, run on those same five images.

| Engine | Size | Avg sim | Avg ms |
|--------|------|--------:|-------:|
| arbo | tiny | 93.3% | ~165 |
| arbo | **small** | **94.4%** | ~530 |
| arbo | medium | 94.8% | ~2120 |
| ppu-paddle-ocr (ref) | small | 94.6% | ~793 |
| rapidocr (ref) | tiny | 92.6% | ~345 |
| rapidocr (ref) | small | 94.5% | ~930 |
| rapidocr (ref) | medium | 94.7% | ~3070 |

!!! warning "Five receipts cannot rank three engines — do not read a winner out of this table"

    At `small` the three land at **94.4%** (arbo), **94.5%** (rapidocr) and
    **94.6%** (ppu-paddle-ocr). The entire spread is ~0.4 pts across five
    images, which is inside the run-to-run noise — at this sample size the
    three engines are **indistinguishable**. If you are choosing between them,
    this table is not the evidence you need.

    The five-receipt set is also an easy one, and it flatters every engine by
    roughly 8–9 pts. Same harness, same ground-truth method, both engines at
    `small`, widened to 40 SROIE receipts:

    | Engine (`small`) | n=5 | n=40 |
    |---|---|---|
    | arbo | 94.2% | 86.1% |
    | rapidocr | 94.5% | 85.3% |

    arbo moves from ~0.1 pt *behind* rapidocr to ~0.8 pt *ahead* of it purely
    by widening the sample: **the ordering is not stable across sample sizes.**
    Treat the n=40 column as the headline and the five-image numbers as the
    smoke test they were built to be.

    (arbo reads 94.2% here rather than the 94.4% above because this pair comes
    from a separate run of the harness — a 0.2 pt swing on byte-identical
    inputs, which is itself a fair measure of how little the five-image spread
    is worth.)

So the reference rows are calibration, not a leaderboard: at `small`, arbo sits
inside half a point of both ports while running faster than either on the same
images, which is what let `small` become the default. The one comparison here
that does survive the sample size is `small` vs `medium` — same engine, same
images, so the noise cancels: **~+0.4 pts for roughly 4× the CPU latency.**

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

## Changes since the accuracy cycle

These landed after the cycle above, from a gap analysis against RapidOCR. No
config default moved, but **both change output**, so they matter if you pin
config or diff results against a stored baseline.

| Change | Old behaviour | New behaviour | Affects |
|---|---|---|---|
| Detector input scaling | `detLimitSideLen` scaled the long side **both ways** | Scale factor clamped to `<= 1.0` — downscale only | Small images: faster, detection may differ |
| Reading-order tolerance | Hard-coded **12px** y-tolerance | Derived from the **median polygon height** of the lines being sorted | High-DPI / unusually-scaled pages: line order may differ |

### `detLimitSideLen` is now a ceiling, not a target

`getScaleParam` used to compute `ratio = detLimitSideLen / longSide` with no
upper guard, so an image *smaller* than the limit was **upscaled** to it. At the
default `detLimitSideLen = 960`, a 200px thumbnail became a 960px detector input
and paid full detection cost for invented pixels.

The ratio is now clamped to `<= 1.0`. `detLimitSideLen` is a maximum and nothing
else — RapidOCR calls this `limit_type: max`, and this is now the same
behaviour. The multiple-of-32 flooring of both dimensions still applies exactly
as before.

!!! warning "Small images now run faster, and their detection results may differ"

    This is not a pure speed win. The detector now sees the image at its
    original resolution instead of an upscaled one, so boxes on sub-`960px`
    inputs can come out differently — usually fewer spurious boxes, but if you
    have a stored baseline for small images, re-generate it. Images already at
    or above `detLimitSideLen` are unaffected: they were being downscaled
    before and are downscaled identically now.

### Reading-order tolerance is now adaptive

`sortLinesReadingOrder` groups lines into visual rows before sorting left to
right within each row. That grouping used a fixed 12px y-tolerance, which is
resolution-dependent: 12px is about half a line height at 100–150 DPI, but on a
300 or 600 DPI scan it is a fraction of one — so rows fragment into single-cell
rows and a two-column row could be emitted in the wrong order.

The tolerance is now derived from the **median polygon height** across the lines
being sorted (roughly half a line height), with a small floor so degenerate
input — one line, or boxes too few to estimate from — still behaves.

!!! warning "Line order on high-DPI pages may differ, and is now scale-invariant"

    Sorting is now independent of page scale: the same page rendered at 1x and
    at 4x produces the same line order. If you captured a `page.lines` ordering
    from a high-DPI or unusually-scaled document, re-check it — the old ordering
    on those pages was the buggy one.

### Memory footprint: the ONNXRuntime CPU arena is off

Not an accuracy change, and it **does not affect output** — but anyone
benchmarking will see it. All three sessions (det, cls, rec) disable the
ONNXRuntime CPU memory arena **by default**. The arena never returns memory to
the OS; RapidOCR measured **5695.5 MiB peak RSS with it on versus 82.1 MiB with
it off**, a 5618 MiB delta on a *single* inference, in exchange for roughly 13%
inference latency. On a 4 GB Jetson that is the difference between running and
being OOM-killed, so the memory is the better trade. Expect slightly higher
per-inference latency and dramatically lower RSS than before.

Opt back in when RAM is plentiful and latency matters:
`EngineConfig::enableCpuMemArena = true`, or `--enable-cpu-mem-arena` on the
CLI. With the flag on, arboOCR matches oar-ocr's arena-on default: −26%
engine latency on the small tier (566→419 ms avg cold-spawn) with
byte-identical OCR output on 40/40 test images — a pure runtime win. See
[Benchmarks](../benchmarks.md#arboocr-vs-oar-ocr).

## v0.4.0: recognition batching, and output that now moves

v0.4.0 is a recognition-performance release. Unlike the arena flag above, it
**does change OCR output**. If you diff against a stored baseline, regenerate it.

| Change | Old behaviour | New behaviour | Affects |
|---|---|---|---|
| Recognition batch strip width | Every batch padded up to a **320px floor** | Sized to the batch's own widest crop (32px minimum floor) | Latency drops sharply; logits for the last characters of a crop can shift |
| CTC decode length | Decoded all `timeSteps` of the padded strip | Decoded only each crop's own share of it | Speed only — the trailing timesteps covered zero padding |
| ORT session flags | Default execution mode, no memory pattern | `ORT_SEQUENTIAL` + `EnableMemPattern` on all three sessions | Speed and RSS only |
| Detector box filter | Every detected box got a crop and an inference | Boxes under `minDetBoxArea` (default `20`, detector-input px²) are dropped first | Dust and speckle no longer produce one-character lines |

### Why the strip width changes output

The batched recognizer forces every crop in a batch to share one width, and it
used to seed that width from the model's *reference* width — `rec_image_shape`'s
320 — growing from there. Nothing required that floor: the crops' own widths are
what the model actually reads. A batch of six ~90px line items was therefore
running the conv stack over 320px of mostly zeros.

The strip is now sized to the widest crop the batch actually holds. That is a
large latency win, but a narrower strip feeds the convolutional stack different
right-edge context, so the logits for the **last few characters of each crop**
move slightly. Measured on a 40-image receipt sample: average similarity
86.31% → 86.40%, but only **8 of 40** stems character-identical (20 better, 12
worse, worst −0.46pp).

!!! warning "This is not the free kind of speed change"

    Compare it to the arena flag above, which was byte-identical on 40/40 images
    and therefore safe to enable blind. This one is not: it is a genuine
    decode-path change with a small, mixed, corpus-dependent effect on output.
    Net flat-to-slightly-up on the sample it was measured on — but **measure on
    your own corpus before adopting it**, and never adopt it in a pipeline that
    diffs against a frozen baseline without regenerating that baseline.

    The unit tests cannot warn you about this. They feed synthetic fixtures
    through the decode, so they pin the decode *algebra* (padding must not move
    content) but never touch real model logits, which is where the difference
    actually appears.

### `minDetBoxArea`

Defaults to `20`, and measured **identical output on 40/40 images** versus
disabled — so for ordinary documents it is not a behaviour change. It stops
pathological detection noise from paying for a crop and a recognition call.

It is expressed in **detector-input** pixels (area, not a side length), so it
scales with `detLimitSideLen` and means the same thing at any source resolution.
Set it to `0` when hunting for genuinely tiny text. See Detection tuning.

### `spaceRecovery`

Off by default, and opt-in for a reason. Recognition models routinely drop
inter-word spaces, so "ATAS NAMA" can come back as "ATASNAMA". With this
enabled, a timestep whose winner is a real character but whose *space* logit is
a strong runner-up emits the space as well.

It is a heuristic resting on a near-miss, which means it can also insert
spurious spaces in dense symbol and number runs where the space class is a
common runner-up. It showed **no benefit on this receipt corpus**, which is part
of why it is off by default: treat it as a targeted fix for a specific
run-together-words problem, not as a setting to flip on and forget.

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
