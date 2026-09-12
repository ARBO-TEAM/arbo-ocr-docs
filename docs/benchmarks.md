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

!!! warning "Batching makes CPU inference slower — by roughly 28% here"
    ~3.9s → ~5.0s is not measurement noise and not a bug. If you deploy
    CPU-only, this is a real cost you are paying, and there is currently no
    flag to opt out of it.

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
arboOCR wins accuracy at every tier (+2.0/+1.6/+1.8 pts tiny/small/medium);
oar-ocr wins engine latency at every tier — but the small-tier gap closed to
near-parity after the ROI-crop fast path and the arena flag:

| Engine (small) | Avg engine ms | Avg sim |
|---|---|---|
| oar small | 401 | 84.7% |
| arbo small, arena off | 566 | 86.3% |
| arbo small, arena on (`--enable-cpu-mem-arena`) | 419 | 86.3% |

Arena on vs off is byte-identical output on 40/40 images — a pure runtime win
(−26% engine) that matches oar's arena-on default. oar's remaining edge is ~5%
engine time, which is noise territory on cold spawn, while arbo keeps +1.6 pts
accuracy. Same advice as ever on sizes: small→medium buys ~+0.3–0.4 pts for
~4× latency on either engine.

Methodology notes: CPU only, one machine; n=40, single run, no thermal
control — treat single-digit-percent latency deltas as noise. Similarity is
character-level Levenshtein after lowercasing and whitespace collapse, not
official SROIE metrics — ranking only. Both arms pay spawn + model reload per
image, so neither engine is shown at warm steady-state.

## See also

- [Accuracy defaults](models/accuracy-defaults.md) — the accuracy measurements, and why the defaults are what they are.
- [Model sizes](models/sizes.md) — `tiny` / `small` / `medium` and what each costs.
- [API Reference](api/index.md#arboocrengine) — `recBatchNum`, `useFp16`, `trtCacheDir`, `backend()`.
