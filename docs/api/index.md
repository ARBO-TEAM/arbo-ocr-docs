---
title: API Reference
---

# API Reference

Seven headers. Most integrations only ever include the first one.

```cpp
#include <arboOCR/engine.hpp>       // Engine, EngineConfig — start here
#include <arboOCR/detector.hpp>     // Detector — text region detection
#include <arboOCR/classifier.hpp>   // Classifier — orientation (0°/180°)
#include <arboOCR/recognizer.hpp>   // Recognizer — CRNN text recognition
#include <arboOCR/visualize.hpp>    // drawResult — outline boxes on a copy
#include <arboOCR/logging.hpp>      // optional log callback (default: silent)
#include <arboOCR/model_downloader.hpp>
```

On this page: the `Engine` facade and its config, the three ways to feed it an
image, the result types, the JSON schema, and the behavioural guarantees you
need to know before you wire this into a service. The stage-level classes are
covered in [Building a custom pipeline](custom-pipeline.md); the debug overlay
in [Visualization](visualize.md).

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
    int         intraOpNumThreads = 0;    // ORT thread pools, all three sessions
    int         interOpNumThreads = 0;    // 0 = ORT default (machine-sized)
    bool        useClahe     = false;     // CLAHE contrast boost before detection (faded/low-contrast docs)
    bool        splitOvermerged = false; // ink-gap split of wide fused det boxes (opt-in)
    float       minimumConfidence = 0.5f; // drop low-conf lines (0 = keep all)
    bool        returnWordBoxes = false;  // per-word polygons in LinePrediction::words
    std::string trtCacheDir  = "models/trt_engines";
    bool        autoDownload = true;      // fetch missing stock weights instead of throwing
    std::string modelsBaseUrl;            // empty = defaultModelsBaseUrl()
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
    PagePrediction recognizeEncoded(const uint8_t* data, size_t size);  // PNG/JPEG bytes
    std::future<PagePrediction> recognizeAsync(const std::string& imagePath);
    std::future<PagePrediction> recognizeAsync(const cv::Mat& image);
    std::future<PagePrediction> recognizeEncodedAsync(const uint8_t* data, size_t size);
};

ModelPaths resolveModelPaths(const EngineConfig& cfg);  // pure — derives paths, touches nothing
ModelPaths ensureOcrModels(const EngineConfig& cfg);    // derives, then fetches what is missing
cv::Mat decodeImageBytes(const uint8_t* data, size_t size);
```

`backend()` reports what actually got selected — `"tensorrt"`, `"cuda"` or
`"cpu"` — not what you asked for. Log it at startup; a config that requested
TensorRT but silently fell back to CPU is the single most common cause of
"why is this 10× slower than the benchmark".

!!! warning "`useCuda` / `useTensorrt` need a v0.3.0 or newer release archive"
    Release archives before **v0.3.0** shipped no
    `onnxruntime_providers_shared` library, so ONNX Runtime could not bring up
    either GPU provider from a published package and `backend()` returned
    `"cpu"` no matter what the config asked for. The provider libraries are
    `dlopen`'d rather than linked, so they appeared in no import table and
    `ldd` could not flag them as missing — the failure had no symptom other
    than the backend string. v0.3.0 is the first release that ships them. If
    you build arboOCR yourself, the same rule applies to whatever you copy
    alongside the binary.

### `ensureOcrModels`: the constructor's first step, exposed

`Engine`'s constructor now calls `ensureOcrModels(cfg)` where it used to call
`resolveModelPaths(cfg)`. It resolves the same four paths and then fills in
whatever is missing, one file at a time, stopping at the first hit:

1. An explicitly set `detModelPath` / `clsModelPath` / `recModelPath` /
   `dictPath` is returned exactly as given and is **never** replaced by a
   download.
2. Otherwise, an existing non-empty file under `modelsDir` wins — a populated
   models directory means zero network access.
3. Otherwise the stock file is downloaded into the per-user cache, checked
   against a SHA-256 compiled into the binary, and that cache path is returned.

`clsModelPath` is only fetched when `useAngleCls` is set, and a dictionary that
cannot be fetched is not fatal — most recognizers carry their charset in their
own ONNX metadata. Set `autoDownload = false` (or export `ARBOOCR_OFFLINE=1`)
to keep step 3 from ever running.

!!! danger "An explicit path is never substituted by a download"
    Rule 1 exists to protect fine-tuned weights. If `recModelPath` names your
    own recognizer and that file is missing, arboOCR does not quietly fetch the
    stock model and hand you plausible output from a network you did not
    choose. The path comes back exactly as you set it and construction fails on
    it — the same failure you would have got before any of this existed.
    Substitution would be invisible from the results, which is what makes it
    the wrong default.

!!! note "`ensureOcrModels` never throws"
    Anything it cannot fetch keeps its resolved-but-missing path, so a failed
    download surfaces as the ordinary model-load error at construction rather
    than as a second, separate failure mode you would have to catch
    differently. `resolveModelPaths` is unchanged and still pure: no
    filesystem check, no network, same answer every time. Call that one when
    you want to know *which* paths a config names; call `ensureOcrModels` when
    you want the download to happen at a moment you picked rather than inside
    the constructor.

### Thread pools: intraOp and interOp

Two `int` fields, both defaulting to `0`, applied to **all three ONNX Runtime
sessions** (detector, classifier, recognizer) at `Engine` construction. In
Python they are `intra_op_num_threads` and `inter_op_num_threads`. Negative
values are clamped to `0` — ORT reads these as a literal thread count, so a
stray `-1` is not the "auto" it looks like.

!!! warning "This is a deployment knob, not a performance win"

    Do not reach for these expecting speed. `0` means "let ORT decide", which
    in practice sizes the pools for **the whole machine** — and that is the
    right answer for a process that owns the box. It is the wrong answer when
    the process does not own the box:

    - **N workers on one host.** Each process independently spawns a
      machine-sized pool. Eight workers on eight cores means eight pools of
      eight threads competing for the same eight cores, and they thrash.
    - **A container with a CPU quota.** A cgroup limit is invisible to ORT: it
      sizes against the host's core count, not your `--cpus` budget, then gets
      throttled.

    Both cases are fixed by *lowering* the value, not raising it. One worker
    per core with `intraOpNumThreads = 1` beats eight machine-sized pools.

RapidOCR exposes the same two knobs and states outright that bigger is not
better — the optimum is workload-dependent, which is precisely why this is
tunable rather than tuned. There is no value arboOCR could ship that would be
correct for a Jetson, a 64-core server and a 0.5-CPU container at once, so the
default stays at ORT's own choice and the decision is handed to whoever knows
the deployment. **Measure on your own hardware and your own images before you
change either field**; if you are running a single OCR process on a dedicated
machine, `0` is already right and you should leave it alone.

!!! note "`intraOpNumThreads` is the one that does anything today"

    Intra-op threads parallelize work *inside* a single operator and are what
    actually determines how many cores one inference burns. Inter-op threads
    parallelize *across* independent operators, which ONNX Runtime only does in
    parallel execution mode — and arboOCR never calls `SetExecutionMode`, so
    every session runs in ORT's default sequential mode. `interOpNumThreads` is
    exposed for symmetry with RapidOCR and for provider configurations that
    schedule differently; set `intraOpNumThreads` if you want to cap CPU usage.

These fields are **library-level only** — there are no `arboocr_demo` flags for
them, so they are reachable from C++ and Python but not from the CLI or the
wrappers that spawn it. If you are constraining a subprocess-based deployment,
constrain it from outside: `taskset`, `cpuset` cgroups, or `--cpus` on the
container.

!!! tip "Consistent with the arena decision"

    arboOCR takes a deliberately conservative stance on ONNX Runtime resource
    defaults. The CPU memory arena is disabled for a closely related reason —
    ORT's default is tuned for a process that owns the machine, and it never
    returns memory to the OS. See
    [Memory footprint: the ONNXRuntime CPU arena is off](../models/accuracy-defaults.md#memory-footprint-the-onnxruntime-cpu-arena-is-off).

### Three ways in: path, `cv::Mat`, encoded bytes

| Input | Call | Use it when |
|---|---|---|
| File path | `recognize(const std::string&)` | The image is already on disk and you want `page.image` populated with the filename. |
| Decoded pixels | `recognize(const cv::Mat&)` | You already hold a BGR frame — camera capture, a page you rasterized yourself, output of your own preprocessing. |
| Encoded bytes | `recognizeEncoded(const uint8_t*, size_t)` | You hold a still-encoded PNG/JPEG: an HTTP multipart upload, a socket read, a blob from object storage, an FFI buffer. |

`recognizeEncoded()` exists so web and API pipelines stop writing a temp file
per request purely to hand the library bytes they already have in memory. That
temp file is not free — it is a syscall pair, a disk write, a cleanup path, and
a race condition per request under concurrency.

```cpp
// std::vector<uint8_t>, std::string, std::span, or a raw FFI buffer — all bind
// to pointer + size with no copy at the boundary.
std::vector<uint8_t> upload = readRequestBody();
auto page = engine.recognizeEncoded(upload.data(), upload.size());
```

It is named `recognizeEncoded` rather than a third `recognize()` overload
deliberately: a bytes overload sitting next to `recognize(const cv::Mat&)`
reads as "raw pixels" to everyone who skims the call site, and the two mean
very different things.

`decodeImageBytes()` is the same decode step exposed as a free function, for
when you want the `cv::Mat` for your own reasons (thumbnailing, dimension
checks, your own preprocessing) before handing it to `recognize()`. It is a
free function rather than an `Engine` member so the contract is testable
without ONNX files on disk.

!!! note "`page.image` is empty for byte and `cv::Mat` input"
    A byte buffer has no filename, so `PagePrediction::image` is left empty and
    the JSON emits `"image":""`. Same for the `cv::Mat` overload. If you need
    to correlate results with a request or document ID, carry it yourself —
    the library will not invent one.

## Behaviour you must know

!!! note "`recognize()` never throws"
    A missing/unreadable image or an inference error degrades to an
    empty-lines result with `elapsedMs` still set. There is no exception to
    catch and no error code to read — check `page.lines.empty()` for that
    case. An empty page is indistinguishable from a genuinely blank image,
    which is deliberate: the caller decides what "no text" means.

    `recognizeEncoded()` holds the same contract against hostile input: a null
    pointer, a zero size, a buffer larger than `INT_MAX`, and undecodable
    garbage all degrade to the same empty-lines page. `decodeImageBytes()`
    returns an empty `cv::Mat` for those four cases. This is load-bearing, not
    defensive noise — `cv::imdecode` asserts (and throws) on an empty buffer
    rather than returning empty, so an unguarded pass-through would crash on
    an empty upload.

!!! danger "Async variants are not safe for concurrent use on the same `Engine`"
    One outstanding call at a time, or one `Engine` per worker. Firing two
    `recognizeAsync()` calls at the same `Engine` and waiting on both futures
    is undefined behaviour, not a slow path. Pool `Engine` instances if you
    need parallelism — each one holds its own session state.

    Buffer lifetime is not part of that hazard: `recognizeEncodedAsync()`
    **copies** the byte buffer before launching (as the `cv::Mat` overload
    clones the image), so you may free or reuse your buffer the moment the call
    returns. You do not have to keep the upload alive until the future
    resolves.

## Result types

```cpp
struct Point2f { float x; float y; };
using Polygon = std::vector<Point2f>;

struct TokenSpan {
    float begin = 0.0f;
    float end   = 0.0f;
};

struct WordBox {
    Polygon polygon;
    std::string text;
    float score = 0.0f;   // mean CTC confidence over this word's characters
};

struct LinePrediction {
    Polygon polygon;
    std::string text;
    float score = 0.0f;      // mean CTC character confidence (0 if no chars)
    float detScore = 0.0f;   // detector box score (0 if unknown)
    std::vector<WordBox> words;  // empty unless EngineConfig::returnWordBoxes
};

struct PagePrediction {
    std::string image;
    std::vector<LinePrediction> lines;
    float elapsedMs = 0.0f;
};

std::string toJson(const PagePrediction& page, bool pretty = false);
std::string toJson(const PagePrediction& page, const std::string& backend, bool pretty = false);
std::string toJson(const LinePrediction& line, bool pretty = false);
```

!!! warning "`score` changed meaning — read this if you are upgrading"
    `LinePrediction.score` is now the **recognizer's mean CTC confidence**. A
    separate `detScore` holds the detector box score (`detScore` in JSON,
    `det_score` in the Python binding). If your code treated `score` as
    detector confidence, switch to `detScore` / `det_score`. See
    [Accuracy defaults](../models/accuracy-defaults.md) for the full list of
    breaking changes in that cycle.

`page.lines` is sorted top→bottom, left→right (centroid y then x) — **not**
raw detector order. Anything that matched boxes by index against an old dump
must use polygon geometry instead.

`Polygon` is a `std::vector<Point2f>` in **source-image** coordinates, using
arboOCR's own `Point2f` rather than OpenCV's so callers do not need OpenCV
types at the API boundary.

### Word boxes

Set `EngineConfig::returnWordBoxes = true` and every `LinePrediction` also
carries `words`: one `WordBox` per word for space-delimited scripts, one per
character for CJK, which has no space to split on. This mirrors RapidOCR's
`return_word_box` behaviour.

```cpp
arbo::ocr::EngineConfig cfg;
cfg.returnWordBoxes = true;
arbo::ocr::Engine engine(cfg);

for (const auto& line : engine.recognize("receipt.jpg").lines) {
    for (const auto& w : line.words) {
        // w.polygon — source-image coordinates
        // w.text, w.score
    }
}
```

It is off by default for a reason worth stating: the `TokenSpan` values it is
built from are nearly free to compute, but *carrying* a polygon for every word
of every line of every page is not, and most callers only want line-level
output.

`TokenSpan` is the underlying primitive — the horizontal extent of one CTC
token as fractions of the recognizer crop's *content* width, so batch padding
is already divided out. `begin`/`end` are in `[0,1]` with `begin <= end`. You
only meet it directly on `RawTextLine` when driving the
[recognizer stage yourself](custom-pipeline.md); `words` is the resolved,
source-image-coordinate form.

!!! warning "Word polygons are approximate — do not crop glyphs from them"
    The polygons are derived from **CTC timestep alignment**, which is a
    by-product of recognition, not a supervised output. CTC peaks partway
    through a glyph rather than at its edges, so boxes run roughly **half a
    character wide of true glyph extents** — measured against synthetic text,
    **consistently lagging right**.

    That is good enough for highlighting, search hit-marking, and reading-order
    reconstruction. It is **not** good enough to crop a glyph out of the source
    image, feed a downstream classifier, or drive layout analysis that assumes
    tight bounds. Widening each span to meet its neighbours would recover the
    extents, but picking that constant needs labelled data arboOCR does not
    have — so the raw alignment is reported honestly instead of being fudged.

## JSON output

`toJson()` never throws. The `backend` overload exists because
`Engine::backend()` is not part of `PagePrediction`, and callers who want it in
the payload (the CLI's `--json` does) should not have to splice it in
themselves. Floats are written with 6 significant digits, so `0.9f` serializes
as `0.9`.

```json
{
  "backend": "cpu",
  "image": "receipt.jpg",
  "elapsedMs": 184.2,
  "lines": [
    {
      "text": "TOTAL 12.50",
      "score": 0.982,
      "detScore": 0.914,
      "polygon": [
        {"x": 42, "y": 118},
        {"x": 233, "y": 118},
        {"x": 233, "y": 146},
        {"x": 42, "y": 146}
      ],
      "words": [
        {"text": "TOTAL", "score": 0.991, "polygon": [{"x": 42, "y": 118}, {"x": 130, "y": 118}, {"x": 130, "y": 146}, {"x": 42, "y": 146}]},
        {"text": "12.50", "score": 0.973, "polygon": [{"x": 140, "y": 118}, {"x": 233, "y": 118}, {"x": 233, "y": 146}, {"x": 140, "y": 146}]}
      ]
    }
  ]
}
```

| Key | Type | Notes |
|---|---|---|
| `backend` | `string` | Only in the `backend` overload. `"tensorrt"` \| `"cuda"` \| `"cpu"`. |
| `image` | `string` | The source path — **`""`** for `cv::Mat` and `recognizeEncoded()` input. |
| `elapsedMs` | `number` | Wall-clock time for the pass. Set even on a failed/empty page. |
| `lines` | `array` | Sorted top→bottom, left→right. Empty on any failure. |
| `lines[].text` | `string` | Decoded text, JSON-escaped (control chars as `\uXXXX`). |
| `lines[].score` | `number` | Mean CTC character confidence. |
| `lines[].detScore` | `number` | Detector box score. Note the camelCase — the Python binding calls it `det_score`. |
| `lines[].polygon` | `array` | `[{"x": …, "y": …}, …]` in source-image coordinates. |
| `lines[].words` | `array` | **Present only when non-empty.** `[{"text": …, "score": …, "polygon": […]}, …]`. |

!!! note "`words` is emitted only when non-empty"
    A line with no word boxes omits the key entirely rather than writing
    `"words": []`. The JSON shape is therefore byte-identical to the previous
    release for every caller who never sets `returnWordBoxes` — existing
    parsers and wrappers keep working untouched. If you consume `words`, treat
    a missing key and an empty array as the same thing.

    In `pretty` mode, line polygons are expanded one point per line but each
    **word** polygon stays on a single line. Word boxes are numerous and a
    fully expanded dump is unreadable; this keeps the pretty output skimmable.

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
| `intraOpNumThreads` | `int` | `0` | ORT intra-op pool size for all three sessions. `0` = ORT decides (sizes for the whole machine). Lower it when several workers or a CPU-quota'd container share a host. Negatives clamp to `0`. See [Thread pools](#thread-pools-intraop-and-interop). |
| `interOpNumThreads` | `int` | `0` | ORT inter-op pool size. `0` = ORT decides. Largely inert today — arboOCR runs sessions in ORT's default sequential execution mode. Negatives clamp to `0`. |
| `useClahe` | `bool` | `false` | CLAHE contrast boost applied to the full image before detection, for faded/low-contrast docs. See [CLAHE](../models/clahe.md). |
| `splitOvermerged` | `bool` | `false` | Opt-in ink-gap split of wide fused detector boxes. See [Accuracy defaults](../models/accuracy-defaults.md). |
| `minimumConfidence` | `float` | `0.5f` | Drops low-confidence lines; `0` keeps every box. Also the green/red threshold in [`drawResult`](visualize.md). See [Accuracy defaults](../models/accuracy-defaults.md). |
| `returnWordBoxes` | `bool` | `false` | Populates `LinePrediction::words` with a polygon per word (per character for CJK). Off by default — the spans are cheap, carrying them for every line of every page is not. Read the [accuracy caveat](#word-boxes) before you rely on the geometry. |
| `trtCacheDir` | `std::string` | `"models/trt_engines"` | Where TensorRT caches built engines. Changing `useFp16` or `recBatchNum` can invalidate this cache — see [Benchmarks](../benchmarks.md#tensorrt-precision-fp16). |
| `autoDownload` | `bool` | `true` | Fetch missing **stock** weights into the per-user cache instead of failing to construct. An explicitly set `*ModelPath` is never downloaded over. Set `false` — or export `ARBOOCR_OFFLINE=1` — for a build agent or container that must not reach the network. See [Environment](../cli.md#environment). |
| `modelsBaseUrl` | `std::string` | *(empty)* | Directory URL the stock files are fetched from. Empty means `defaultModelsBaseUrl()`: the pinned release, or `ARBOOCR_MODELS_URL` when that is set. Point it at an internal mirror. |
| `modelsDir` | `std::string` | `"models"` | Base directory used to derive every path left empty below. |
| `detModelPath` | `std::string` | *(empty)* | Empty = `modelsDir/ocrVersion_det.onnx`. |
| `clsModelPath` | `std::string` | *(empty)* | Empty = `modelsDir/ocrVersion_cls.onnx`. |
| `recModelPath` | `std::string` | *(empty)* | Empty = `modelsDir/ocrVersion_rec_modelType.onnx`. |
| `dictPath` | `std::string` | *(empty)* | Empty = `modelsDir/ocrVersion_rec_modelType_dict.txt`. |

!!! tip "Check resolution before you construct"
    `resolveModelPaths(cfg)` applies exactly the derivation rules in the last
    five rows and nothing else. Call it first and assert the files exist — you
    get a clear error at your own call site instead of a silent empty-lines
    page later. With `autoDownload = true` that assertion is the wrong check on
    its own, because a missing file is not yet a problem: call
    [`ensureOcrModels(cfg)`](#ensureocrmodels-the-constructors-first-step-exposed)
    instead and assert on what it returns.

## Where the real documentation lives

[`include/arboOCR/`](https://github.com/wafik/ArboOCR/tree/main/include/arboOCR)
carries full doc comments on each class. Every non-obvious design decision is
documented inline where the code lives, not just here — if a default looks
arbitrary, the header explains why it is not.

- [Logging](logging.md) — install a callback; the library is silent by default.
- [Building a custom pipeline](custom-pipeline.md) — `Detector`, `Classifier`, `Recognizer` directly.
- [Visualization](visualize.md) — `drawResult`, the box-overlay debug helper.
- [Model downloader](downloader.md) — `downloadFile` (now with an optional `expectedSha256`), `downloadOcrModels`, `ocrModelFileNames`, plus the defaults behind auto-download: `defaultModelsTag`, `defaultModelsBaseUrl`, `defaultModelsCacheDir`, `sha256File`, `knownSha256`.
