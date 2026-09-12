---
title: Benchmarks
---

# Benchmarks

All numbers measured on a **Jetson Nano**, real inference (not synthetic
timing), on a mix of document types — not just one lucky sample image.

!!! note "This page is latency, not accuracy"
    Character-similarity measurements live in
    [Accuracy defaults](models/accuracy-defaults.md). Don't conflate the two:
    the config that makes arboOCR fastest is not always the one that makes it
    most accurate, and several defaults here were chosen against the accuracy
    numbers, not these.

## Across document types (tiny recognizer, CPU)

| Document | Lines | Latency |
|---|---|---|
| Table/form (bilingual, numeric) | 60 | 1.07s |
| Newspaper (multi-column) | 48 | 1.10s |
| Legal contract (dense paragraphs) | 24 | 1.01s |
| ID card (short fields) | 28 | 0.74s |
| Whiteboard menu (handwriting/marker) | 11 | 0.73s |
| Receipt | 31 | ~0.75s |

No crashes, no garbled output, across layouts ranging from dense tables to
handwritten menus.

Note what the spread says: latency tracks line count far more than layout
complexity. A 60-line bilingual table and a 48-line newspaper cost about the
same; an 11-line whiteboard menu is cheap regardless of how ugly the
handwriting is.

### Batching: CPU vs. TensorRT

Recognition batches up to `recBatchNum` text-line crops per inference call
(default 6). This is a genuine trade-off, not a universal win:

| Backend | Before batching | After batching |
|---|---|---|
| CPU | ~3.9s | ~5.0s (**slower**) |
| TensorRT | ~456ms | ~340–460ms (**faster**) |

!!! warning "These CPU numbers predate v0.4.0 — and the floor behind them is gone"

    The ~3.9s → ~5.0s CPU penalty above was measured before **v0.4.0**, when
    every batch strip was padded to a 320px floor regardless of what it held.
    That floor was the dominant source of the wasted padding, and v0.4.0
    **removed it** in favour of sizing each strip to its own widest crop.

    That does not mean the penalty is gone — crops in a batch still share one
    width, which is the mechanism below — but the number above is a measurement
    of code that no longer exists, on the bundled sample image. **Re-measure on
    your own hardware and corpus before using either figure.**

    What v0.4.0 *did* measure, on x86-64 over a 40-image receipt corpus, is the
    floor change alone: −26% engine latency with batching still on in both
    arms. Batching against no-batching was **not** A/B'd in that pass, so there
    is currently no fresh post-v0.4.0 number for this particular comparison —
    on any architecture, including Jetson Nano.

CPU has no real parallelism across the batch dimension, so padding every
crop to a shared width is wasted computation there. TensorRT/GPU backends
parallelize across the batch dimension, where batching pays off. Batching
is always on — there's currently no flag to disable it for CPU-only
deployments.

The mechanism is worth internalising, because it also tells you how to tune
`recBatchNum`:

=== "CPU"

    No parallelism across the batch dimension. Every crop in a batch is
    padded to a shared width, and the padding is computed for nothing. Larger
    `recBatchNum` means more padding waste, not more throughput.

=== "CUDA / TensorRT"

    The batch dimension is exactly what the GPU parallelizes over. Padding
    cost is amortised across crops that execute concurrently, so raising
    `recBatchNum` pays off until you run out of VRAM — which is why the field
    comment says *raise on GPU VRAM*.

### TensorRT precision (FP16)

When `useTensorrt` is true, TensorRT builds engines with **FP16 enabled by
default** (`EngineConfig::useFp16 = true`). That is the usual edge-device
setting on Jetson / desktop GPUs: lower latency and smaller engines, with
minimal accuracy loss for OCR. Set `useFp16 = false` only if you need FP32
for debugging (expect slower first-run compile and runtime).

!!! warning "Changing precision or batch size can invalidate the engine cache"
    Changing `useFp16` (or `recBatchNum`) can invalidate cached engines under
    `trtCacheDir` — clear that directory or use a separate cache path so TRT
    rebuilds instead of loading a mismatched engine. If you are A/B-ing FP16
    against FP32, give each configuration its own `trtCacheDir`; otherwise
    your "FP32 run" may quietly be a cached FP16 engine and your numbers are
    meaningless.

**INT8** is not exposed: it needs a representative calibration dataset and
is easy to get wrong for multilingual text. Prefer FP16 for now.

```cpp
cfg.useTensorrt = true;
cfg.useFp16 = true;   // default — keep for production edge latency
// cfg.useFp16 = false; // FP32 engines for accuracy A/B only
```

## arboOCR vs oar-ocr

Against [oar-ocr](https://github.com/GreatV/oar-ocr) v0.9.2 (Rust, `ort`
bindings) on a 40-image SROIE2019 stem sample: identical PP-OCRv6 weights and
dicts, matched config (det 0.3/0.5/1.6, limit-side 960 ceiling-only, rec batch
6, conf 0.5), CPU cold-spawn on both arms, char similarity vs box ground truth.
arboOCR wins accuracy at every tier (+2.0/+1.6/+1.8 pts tiny/small/medium).

**Small tier, v0.4.0** — every arm run in one session, so a drift in machine
state cannot be mistaken for a code change:

| Engine (small) | Avg engine ms | Avg wall ms | Avg sim |
|---|---|---|---|
| oar-ocr | 529 | 703 | 84.72% |
| **arboOCR, arena on** | **363** | **559** | **86.40%** |
| arboOCR, arena on, `--min-det-box-area 0` | 364 | 552 | 86.40% |
| ppu-paddle-ocr | 489 | 956 | 83.51% |

oar-ocr and ppu-paddle-ocr were re-run from **byte-identical binaries** as
controls and reproduced their previous numbers to the digit (0/40 stems changed
similarity), which is what lets arboOCR's own drop be attributed to the code
rather than to a warm machine. That before/after pair is measured on the same
sample, same session, same harness:

| arboOCR small, arena on | Avg engine ms | Avg sim |
|---|---|---|
| v0.3.0 | 493 | 86.31% |
| v0.4.0 | **363** | **86.40%** |

!!! note "`566 → 419 ms` is a different pair from `493 → 363 ms`"

    Both pairs are real, both are 40-image runs, and they are **not** two steps of
    one improvement — the two `−26%` figures are a coincidence of magnitude.

    - **`566 → 419 ms`** was the arena flag's own A/B, measured *within a single
      session* (arena off vs arena on, otherwise identical binary).
    - **`493 → 363 ms`** is v0.4.0's change, and its controls are oar-ocr and
      ppu-paddle-ocr re-run byte-identically in each session.

    Note that the same configuration — arboOCR small, arena on — reads **419 ms**
    in the first session and **493 ms** in the second. That is not a contradiction:
    in the first session **oar-ocr itself ran 401 ms** versus 529 ms in the second,
    so that whole session was roughly 30% faster and both engines moved together.
    Cold-spawn numbers are only comparable inside one session, which is why the
    table above re-runs its controls rather than quoting them.

!!! warning "v0.4.0 changed output — do not read the similarity delta as a gain"

    Similarity moved 86.31% → 86.40%, but only **8 of 40** stems are
    character-identical: 20 improved, 12 got worse, worst −0.46pp. Batch strips
    are no longer padded to a 320px floor, which changes the logits for the last
    characters of a crop — this is a real behaviour change, not a measurement
    artifact. Net flat-to-slightly-up here; **re-measure on your own corpus.**

!!! note "The three engines bundle different onnxruntime versions"

    arboOCR 1.23.2 (the vcpkg default), oar-ocr 1.28.0, ppu-paddle-ocr 1.29.0 via
    `onnxruntime-node`. These were **not** equalised, so part of the remaining
    latency distance is plausibly the runtime rather than either engine's code.
    arboOCR can opt into 1.28.0 with `-DARBOOCR_ORT_VERSION=1.28.0`.

Same advice as ever on sizes: small→medium buys ~+0.3–0.4 pts for ~4× latency
on either engine.

Methodology notes: CPU only, one machine; n=40, single run, no thermal
control — treat single-digit-percent latency deltas as noise. Similarity is
character-level Levenshtein after lowercasing and whitespace collapse, not
official SROIE metrics — ranking only. Both arms pay spawn + model reload per
image, so neither engine is shown at warm steady-state. `minDetBoxArea`
produced identical output on 40/40 images at its default versus disabled.

## See also

- [Accuracy defaults](models/accuracy-defaults.md) — the accuracy measurements, and why the defaults are what they are.
- [Model sizes](models/sizes.md) — `tiny` / `small` / `medium` and what each costs.
- [API Reference](api/index.md#arboocrengine) — `recBatchNum`, `useFp16`, `trtCacheDir`, `backend()`.
