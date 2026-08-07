---
title: FAQ
description: >-
  Short answers to the questions that come up most often when integrating
  arboOCR — models, defaults, threading, and performance.
---

# FAQ

Short answers with a link to the page that has the full one.

??? question "Which model files do I actually need?"

    Four filenames live under `modelsDir`, but you rarely need all of them:

    - `PP-OCRv6_det.onnx` — text detection. **One file, no size variants.**
      `modelType` does not select a detector.
    - `PP-OCRv6_rec_<size>.onnx` — recognition, one per size
      (`tiny` | `small` | `medium`).
    - `PP-OCRv6_rec_<size>_dict.txt` — character dict, **only** if the
      character list is not embedded in the ONNX metadata.
    - `PP-OCRv6_cls.onnx` — angle classification, **only** needed if
      `useAngleCls` is on.

    See [Models](models/index.md) for the directory layout and the two ways
    to obtain the files.

??? question "Why is there no default model download URL?"

    Because PP-OCR model hosting locations are not stable across mirrors, so
    arboOCR makes the caller pick the source rather than baking in a URL that
    rots. `downloadOcrModels()` takes a `baseUrl` you supply; there is no
    built-in default.

    See [Models](models/index.md) and the
    [model downloader](api/downloader.md).

??? question "Should I use tiny, small or medium?"

    `small` is the default and the right starting point on CPU. On the SROIE
    smoke set (warm Python `Engine`, CPU, 5 receipts):

    | modelType | Size | CPU latency | Full-page sim |
    |---|---|---|---|
    | `tiny` | 4.3 MB | ~165 ms | ~93.3% |
    | `small` (**default**) | ~20 MB | ~530 ms | ~94.4% |
    | `medium` | 73 MB | ~2120 ms | ~94.8% |

    Medium costs roughly **4x the CPU latency for ~+0.4 pts**. Prefer `tiny`
    for throughput; reach for `medium` only with GPU/TensorRT headroom *and* a
    measured win on your own data.

    See [Model sizes](models/sizes.md).

??? question "Does it support language X?"

    Probably, without any configuration. The default PP-OCRv6 recognition
    models are **already multi-language in a single ONNX file** — medium and
    small cover 50 languages including Chinese, English, Japanese and 46
    Latin-script languages; tiny is similar but without Japanese. arboOCR has
    **no `language` config**: load the default models and you get that
    coverage.

    For scripts outside the model's training set, point `recModelPath` (and
    `dictPath`, if your ONNX has no embedded character list) at your own files.

    See [Languages](models/languages.md).

??? question "Why did my line order change after upgrading?"

    `page.lines` is now **always sorted by centroid — y first, then x** —
    rather than emitted in raw detector order. Full-page string metrics and
    human reading both depend on order, so the sort is unconditional.

    Anything that assumed detector emission order, or that matched boxes by
    index against an older dump, needs to use polygon geometry or re-sort the
    same way.

    See [Accuracy defaults](models/accuracy-defaults.md).

??? question "Why are some lines missing from the output?"

    `minimumConfidence` defaults to **`0.5`** and drops low-confidence lines
    (PaddleOCR's `drop_score`). Low-contrast logos, rules read as `+-`, and
    weak boxes disappear from `page.lines`.

    Set `minimumConfidence = 0` to keep every box — that is the right setting
    for audit dumps. Pure symbol lines may need it raised to `0.8` instead.

    See [Accuracy defaults](models/accuracy-defaults.md).

??? question "Does `recognize()` throw?"

    **Never.** A missing or unreadable image, or an inference error, degrades
    to an empty-lines result with `elapsedMs` still set. Check
    `page.lines.empty()` to detect that case — there is no exception to catch.

    Log callbacks are equally defensive: a callback that throws is swallowed,
    so a broken sink cannot crash OCR.

    See the [API reference](api/index.md) and [logging](api/logging.md).

??? question "Can I call `recognizeAsync()` concurrently on one Engine?"

    No. The async variants are **not safe for concurrent use on the same
    `Engine`** — keep one outstanding call at a time, or give each worker its
    own `Engine`.

    See the [API reference](api/index.md).

??? question "Why is batched recognition slower on my CPU?"

    Because CPU has **no real parallelism across the batch dimension**, so
    padding every crop to a shared width is wasted computation. Measured on
    one page: CPU went from ~3.9s to ~5.0s *with* batching, while TensorRT
    went from ~456 ms to ~340–460 ms.

    Recognition batches up to `recBatchNum` crops per inference call (default
    6) and batching is always on — there is currently no flag to disable it
    for CPU-only deployments.

    See [Benchmarks](benchmarks.md).

??? question "Can I get structured output instead of a flat list of lines?"

    Yes — `toMarkdown(page)` reconstructs a **rough** markdown document from
    line geometry: lines that are close together and left-aligned merge into
    one paragraph, taller-than-median lines become `#`/`##` headings, bullet
    and numbered lines stay list items, and a line split by one wide internal
    word gap becomes a `key | value` table row. The CLI writes it with
    `--markdown <path>` (which implies `--word-boxes`); Python has
    `to_markdown(page)`.

    It is a heuristic, not layout analysis: **no multi-column splitting, no
    table-grid reconstruction, no image or rule detection**, and it degrades
    quietly on skewed text rather than failing loudly.

    See [Markdown export](api/markdown.md).

??? question "How do I use arboOCR from Python?"

    Via the pybind11 `Engine` facade, which runs on the same native backends
    as C++. It is **off by default** (`ARBOOCR_BUILD_PYTHON=OFF`) — enable it
    when you have Python 3 development headers.

    Caveats for v1: **no published wheels**, so you build the extension
    against your local `arboOCR` library; NumPy images must be **HxWx3 uint8
    BGR** (OpenCV layout); and there are no async or low-level
    `Detector`/`Recognizer` bindings. If you just want text out of an image,
    the bundled CLI (`arboocr_demo --image page.jpg --models-dir models`)
    needs no bindings at all.

    See [Wrappers](wrappers/index.md) and [Python](wrappers/python.md).
