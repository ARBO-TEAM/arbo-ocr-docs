---
title: Architecture
---

# Architecture

## The pipeline

`Engine::recognize()` is a facade over three independent stages. One call in,
one `PagePrediction` out.

```text
your app
   │
   ▼
┌─────────────────────────────────────────────┐
│  Engine::recognize(imagePath)               │
│                                             │
│   1. Detector::getTextBoxes()   — find lines│
│   2. Classifier::getAngles()    — 0°/180°?  │
│   3. Recognizer::getTextLines() — read text │
│      (batched, up to 6 crops/call)          │
│                                             │
└─────────────────────────────────────────────┘
   │
   ▼
PagePrediction { lines[], elapsedMs }
```

Each stage owns its ONNXRuntime session independently and can be used on its
own. `ocr_utils.hpp` holds the shared geometry/normalization helpers (box
scoring, perspective crop, mean/norm) all three stages build on.

That independence is the point, not an implementation detail:

| Stage | Type | What it does |
|---|---|---|
| 1 | [`Detector`](api/index.md) | Finds text line regions in the page |
| 2 | [`Classifier`](api/index.md) | Decides 0° vs. 180° orientation per crop |
| 3 | [`Recognizer`](api/index.md) | Reads text from crops, batched up to 6 per call |

If you only need boxes, construct a `Detector` and never pay for the other two
sessions. If you already have crops from somewhere else, feed a `Recognizer`
directly. Driving the stages yourself instead of going through `Engine` is
covered in [Building a custom pipeline](api/custom-pipeline.md); the facade and
its configuration are in the [API Reference](api/index.md).

!!! note "Why the shared helpers live in one header"

    All three stages need the same geometry and normalization primitives — box
    scoring, perspective crop, mean/norm. Keeping them in `ocr_utils.hpp`
    rather than duplicating them per stage is what makes a standalone
    `Recognizer` produce byte-identical preprocessing to the one inside
    `Engine`. Mismatched preprocessing between a custom pipeline and the facade
    is the classic source of "my scores dropped and I changed nothing".

## Repository layout

```text
arboOCR/
├── include/arboOCR/     public headers — this is the whole API surface
├── src/arboOCR/         implementation
├── cli/                 arboocr_demo — reference CLI usage
├── examples/            basic_recognize, custom_pipeline — buildable, runnable
├── tests/               doctest suite (buildable, runnable via ctest)
├── vendor/              Clipper (vendored) + onnxruntime (user-provisioned)
└── docs/                verification notes from real hardware testing
```

A few of these are worth calling out:

- **`include/arboOCR/`** is the entire API surface. If a type is not in there,
  it is not something you are meant to depend on.
- **[`examples/`](https://github.com/wafik/ArboOCR/tree/main/examples)** are
  buildable, runnable programs — `basic_recognize` and `custom_pipeline` — not
  snippets that drift out of date because nobody compiles them.
- **`vendor/`** mixes vendored and user-provisioned dependencies: Clipper ships
  in the repo, onnxruntime does not. On aarch64 you populate it yourself — see
  [Jetson / aarch64](build/jetson.md).
- **`docs/`** in the source repo holds verification notes from real hardware
  testing, which is where the [benchmark](benchmarks.md) numbers come from.
