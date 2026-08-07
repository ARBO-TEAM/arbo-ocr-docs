---
title: API Reference
---

# API Reference

Six headers. Most integrations only ever include the first one.

```cpp
#include <arboOCR/engine.hpp>       // Engine, EngineConfig — start here
#include <arboOCR/detector.hpp>     // Detector — text region detection
#include <arboOCR/classifier.hpp>   // Classifier — orientation (0°/180°)
#include <arboOCR/recognizer.hpp>   // Recognizer — CRNN text recognition
#include <arboOCR/logging.hpp>      // optional log callback (default: silent)
#include <arboOCR/model_downloader.hpp>
```

On this page: the `Engine` facade and its config, the result types, and the
behavioural guarantees you need to know before you wire this into a service.
The stage-level classes are covered in
[Building a custom pipeline](custom-pipeline.md).

### `arbo::ocr::Engine`

The facade. Construct once with an `EngineConfig`, call `recognize()` per
image.

```cpp
struct EngineConfig {
    std::string ocrVersion   = "PP-OCRv6";
    std::string modelType    = "small";   // recognizer size: tiny | small | medium
    float       detBoxThresh = 0.5f;
    float       detThresh    = 0.3f;
    float       detUnclipRatio = 1.6f;
    int         detLimitSideLen = 960;   // det longest side (960 beat 1536 on SROIE smoke)
    int         recBatchNum  = 6;         // crops per rec inference (raise on GPU VRAM)
    bool        useAngleCls  = false;
    bool        useCuda      = false;
    bool        useTensorrt  = false;
    bool        useFp16      = true;      // TensorRT only — FP16 kernels (default on)
    bool        useClahe     = false;     // CLAHE contrast boost before detection (faded/low-contrast docs)
    bool        splitOvermerged = false; // ink-gap split of wide fused det boxes (opt-in)
    float       minimumConfidence = 0.5f; // drop low-conf lines (0 = keep all)
    std::string trtCacheDir  = "models/trt_engines";
    std::string modelsDir    = "models";
    std::string detModelPath;   // empty = modelsDir/ocrVersion_det.onnx
    std::string clsModelPath;   // empty = modelsDir/ocrVersion_cls.onnx
    std::string recModelPath;   // empty = modelsDir/ocrVersion_rec_modelType.onnx
    std::string dictPath;       // empty = modelsDir/ocrVersion_rec_modelType_dict.txt
};

class Engine {
public:
    explicit Engine(const EngineConfig& config);
    std::string backend() const;                       // "tensorrt" | "cuda" | "cpu"
    PagePrediction recognize(const std::string& imagePath);
    PagePrediction recognize(const cv::Mat& image);    // in-memory (no disk I/O)
    std::future<PagePrediction> recognizeAsync(const std::string& imagePath);
    std::future<PagePrediction> recognizeAsync(const cv::Mat& image);
};

ModelPaths resolveModelPaths(const EngineConfig& cfg);
```

`backend()` reports what actually got selected — `"tensorrt"`, `"cuda"` or
`"cpu"` — not what you asked for. Log it at startup; a config that requested
TensorRT but silently fell back to CPU is the single most common cause of
"why is this 10× slower than the benchmark".

## Behaviour you must know

!!! note "`recognize()` never throws"
    A missing/unreadable image or an inference error degrades to an
    empty-lines result with `elapsedMs` still set. There is no exception to
    catch and no error code to read — check `page.lines.empty()` for that
    case. An empty page is indistinguishable from a genuinely blank image,
    which is deliberate: the caller decides what "no text" means.

!!! danger "Async variants are not safe for concurrent use on the same `Engine`"
    One outstanding call at a time, or one `Engine` per worker. Firing two
    `recognizeAsync()` calls at the same `Engine` and waiting on both futures
    is undefined behaviour, not a slow path. Pool `Engine` instances if you
    need parallelism — each one holds its own session state.

## Result types

```cpp
struct LinePrediction { Polygon polygon; std::string text; float score; };
struct PagePrediction  { std::string image; std::vector<LinePrediction> lines; float elapsedMs; };

std::string toJson(const PagePrediction& page, bool pretty = false);
std::string toJson(const LinePrediction& line, bool pretty = false);
```

!!! warning "`score` changed meaning — read this if you are upgrading"
    `LinePrediction.score` is now the **recognizer's mean CTC confidence**. A
    separate `detScore` holds the detector box score (`det_score` in JSON and
    the Python binding). If your code treated `score` as detector confidence,
    switch to `detScore` / `det_score`. See
    [Accuracy defaults](../models/accuracy-defaults.md) for the full list of
    breaking changes in that cycle.

`page.lines` is sorted top→bottom, left→right (centroid y then x) — **not**
raw detector order. Anything that matched boxes by index against an old dump
must use polygon geometry instead.

## `EngineConfig` field reference

| Field | Type | Default | What it does |
|---|---|---|---|
| `ocrVersion` | `std::string` | `"PP-OCRv6"` | Model family; also the filename prefix used when the explicit `*ModelPath` fields are empty. |
| `modelType` | `std::string` | `"small"` | Recognizer size: `tiny` \| `small` \| `medium`. Selects the **recognizer only** — see [Model sizes](../models/sizes.md). |
| `detBoxThresh` | `float` | `0.5f` | Detector box score threshold. |
| `detThresh` | `float` | `0.3f` | Detector binarization threshold. |
| `detUnclipRatio` | `float` | `1.6f` | Detector polygon unclip ratio (how far boxes are expanded from the shrunk map). |
| `detLimitSideLen` | `int` | `960` | Longest side for detector resize. 960 beat 1536 on the SROIE smoke set — bigger is not better. See [Accuracy defaults](../models/accuracy-defaults.md). |
| `recBatchNum` | `int` | `6` | Crops per recognizer inference call; raise on GPU VRAM. Batching is a CPU/GPU trade-off — see [Benchmarks](../benchmarks.md#batching-cpu-vs-tensorrt). |
| `useAngleCls` | `bool` | `false` | Opt-in 0°/180° orientation classifier. Leave off unless your input is actually rotated. |
| `useCuda` | `bool` | `false` | Request the CUDA execution provider. Confirm with `backend()`. |
| `useTensorrt` | `bool` | `false` | Request the TensorRT execution provider. Confirm with `backend()`. |
| `useFp16` | `bool` | `true` | TensorRT only — FP16 kernels, on by default. See [Benchmarks](../benchmarks.md#tensorrt-precision-fp16). |
| `useClahe` | `bool` | `false` | CLAHE contrast boost applied to the full image before detection, for faded/low-contrast docs. See [CLAHE](../models/clahe.md). |
| `splitOvermerged` | `bool` | `false` | Opt-in ink-gap split of wide fused detector boxes. See [Accuracy defaults](../models/accuracy-defaults.md). |
| `minimumConfidence` | `float` | `0.5f` | Drops low-confidence lines; `0` keeps every box. See [Accuracy defaults](../models/accuracy-defaults.md). |
| `trtCacheDir` | `std::string` | `"models/trt_engines"` | Where TensorRT caches built engines. Changing `useFp16` or `recBatchNum` can invalidate this cache — see [Benchmarks](../benchmarks.md#tensorrt-precision-fp16). |
| `modelsDir` | `std::string` | `"models"` | Base directory used to derive every path left empty below. |
| `detModelPath` | `std::string` | *(empty)* | Empty = `modelsDir/ocrVersion_det.onnx`. |
| `clsModelPath` | `std::string` | *(empty)* | Empty = `modelsDir/ocrVersion_cls.onnx`. |
| `recModelPath` | `std::string` | *(empty)* | Empty = `modelsDir/ocrVersion_rec_modelType.onnx`. |
| `dictPath` | `std::string` | *(empty)* | Empty = `modelsDir/ocrVersion_rec_modelType_dict.txt`. |

!!! tip "Check resolution before you construct"
    `resolveModelPaths(cfg)` applies exactly the derivation rules in the last
    five rows. Call it first and assert the files exist — you get a clear
    error at your own call site instead of a silent empty-lines page later.

## Where the real documentation lives

[`include/arboOCR/`](https://github.com/wafik/ArboOCR/tree/main/include/arboOCR)
carries full doc comments on each class. Every non-obvious design decision is
documented inline where the code lives, not just here — if a default looks
arbitrary, the header explains why it is not.

- [Logging](logging.md) — install a callback; the library is silent by default.
- [Building a custom pipeline](custom-pipeline.md) — `Detector`, `Classifier`, `Recognizer` directly.
- [Model downloader](downloader.md) — `downloadFile`, `downloadOcrModels`.
