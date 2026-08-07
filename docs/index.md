---
title: Overview
hide:
  - navigation
---

# arboOCR

**Standalone C++ OCR library — detection, orientation, and recognition, on CPU, CUDA, or TensorRT.**

Drop it into your own project, point it at an image, get text back.

[Get started :material-arrow-right:](quickstart.md){ .md-button .md-button--primary }
[Browse the API](api/index.md){ .md-button }

---

## What is this?

arboOCR runs the classic three-stage OCR pipeline — **detect** text regions,
**classify** their orientation, **recognize** the characters — over PP-OCRv6
ONNX models via ONNXRuntime. It was extracted from a larger benchmarking
harness into a single-purpose library with no baggage: no TTS, no HTTP
server, no dataset tooling, just the inference core.

<div class="grid cards" markdown>

-   :material-chip: **CPU / CUDA / TensorRT, auto-detected**

    ---

    Construct one `Engine` — it picks the fastest backend available and tells
    you which one it picked. With TensorRT, FP16 is on by default
    ([`useFp16`](benchmarks.md#tensorrt-precision-fp16)) for edge latency.

-   :material-view-module: **Facade *and* building blocks**

    ---

    Use [`Engine::recognize()`](api/index.md#arboocrengine) for the common case,
    or drop down to `Detector` / `Classifier` / `Recognizer` directly if you're
    [building a custom pipeline](api/custom-pipeline.md).

-   :material-layers-triple: **Batched recognition**

    ---

    Text-line crops are batched (default 6 per inference call, matching
    PaddleOCR/RapidOCR) instead of one ONNXRuntime call per line. See
    [Benchmarks](benchmarks.md#batching-cpu-vs-tensorrt) for when this
    actually helps — it is a trade-off, not a free win.

-   :material-memory: **In-memory and async**

    ---

    `recognize(const cv::Mat&)` skips disk I/O for camera and API pipelines;
    `recognizeAsync()` returns a `std::future` for non-blocking multi-image
    work.

-   :material-source-branch: **Ported, not reinvented**

    ---

    The detection/recognition math is a near-verbatim port of
    [RapidOcrOnnx](https://github.com/RapidAI/RapidOcrOnnx) (Apache-2.0) —
    battle-tested logic, renamed and reorganized for a clean public API.

-   :material-language-python: **Wrappers for four languages**

    ---

    [Python](wrappers/python.md), [Go](wrappers/go.md),
    [Rust](wrappers/rust.md), and [PHP](wrappers/php.md) wrappers drive the
    prebuilt binary via subprocess — no C++ build required.

</div>

## Where to go next

| If you want to… | Read |
|---|---|
| See working code in 30 seconds | [Quickstart](quickstart.md) |
| Compile the library | [Build](build/index.md) · [Jetson / aarch64](build/jetson.md) |
| Know which model files you need | [Models](models/index.md) |
| Pick `tiny` vs `small` vs `medium` | [Choosing a model size](models/sizes.md) |
| Upgrade an existing integration | [Accuracy defaults](models/accuracy-defaults.md) |
| Read the full type reference | [API Reference](api/index.md) |
| Use it from Python/Go/Rust/PHP | [Wrappers](wrappers/index.md) |
| Know how fast it is | [Benchmarks](benchmarks.md) |

!!! warning "Upgrading from an older build?"

    Several defaults changed in a recent accuracy cycle — line ordering,
    `modelType`, `detLimitSideLen`, and the meaning of `LinePrediction::score`
    are all different now. Read [Accuracy defaults](models/accuracy-defaults.md)
    before you upgrade.

## License

arboOCR is licensed under the [Apache License 2.0](license.md). The ported
detection/recognition logic derives from
[RapidOcrOnnx](https://github.com/RapidAI/RapidOcrOnnx) (Apache-2.0) and bundles
[Clipper](https://www.angusj.com/clipper2/) (Boost Software License 1.0).
