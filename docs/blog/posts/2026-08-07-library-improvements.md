---
date:
  created: 2026-08-07
authors:
  - arbo-team
categories:
  - Release notes
tags:
  - performance
  - api
description: >-
  A gap analysis against RapidOCR: ten changes shipped, four backends declined,
  and every decision anchored to a number somebody already measured.
---

# What RapidOCR measured, and what we changed because of it

RapidOCR publishes something most OCR projects do not: **measured numbers** for
its engine choices and its parameter defaults. Peak RSS with and without the
ONNXRuntime memory arena. CoreML against the CPU provider, per model. CUDA
against CPU on two different GPUs. Not "we chose X because X is fast" — actual
tables.

We read those tables against arboOCR's source. Where RapidOCR had measured
something and arboOCR had guessed, the measurement won. That is the entire
selection criterion for this round of changes, and it is also why four things
that look like obvious wins are **not** in it.

<!-- more -->

## The memory arena was costing 5.6 GB

All three ONNXRuntime sessions — detector, classifier, recognizer — configured
thread counts and the optimization level and then left the CPU memory arena at
ORT's default, which is on. RapidOCR profiled a single inference both ways:

| `enable_cpu_mem_arena` | Peak RSS | Delta on the inference line |
|---|---:|---:|
| `true` (ORT default) | 5695.5 MiB | +5618.3 MiB |
| `false` | 82.1 MiB | +5.3 MiB |

**5618 MB on one inference**, and the arena never gives it back to the OS. The
purchase price of that memory is roughly 13% inference latency.

Jetson Nano is arboOCR's stated target and its benchmark machine. On a 4 GB
Nano, 5.6 GB of resident arena is not a tuning knob — it is the difference
between running and being OOM-killed. The arena is now disabled in all three
sessions, matching RapidOCR's shipped default. Expect marginally slower
inference and dramatically lower RSS.

## Two changes that alter output

Both are behavioural, both are documented in full on
[Accuracy defaults](../../models/accuracy-defaults.md).

**Small images are no longer upscaled.** `getScaleParam` computed
`ratio = detLimitSideLen / longSide` with no upper guard, so an image *smaller*
than the limit got grown to it. At the default `detLimitSideLen = 960`, a 200px
thumbnail became a 960px detector input and paid full detection cost for
invented pixels. The ratio is now clamped to `<= 1.0`, which makes
`detLimitSideLen` a true ceiling — RapidOCR calls this `limit_type: max`. Small
images run faster and their detection results may differ, because the detector
now sees the original resolution.

**Reading-order tolerance is derived, not hard-coded.**
`sortLinesReadingOrder` grouped lines into visual rows with a fixed 12px
y-tolerance. 12px is about half a line height at 100–150 DPI; at 600 DPI it is
a quarter of a *character*, so every row on the page fragments into single-cell
rows in arbitrary x-order. That is not a small bug — the accuracy cycle
measured reading-order sorting alone as worth ~6.7 points of full-page
character similarity. The tolerance now comes from the median polygon height
across the lines being sorted, with a floor for degenerate input. Sorting is
consequently scale-invariant: the same page at 1x and at 4x sorts identically.

## The documented download path produced a broken models directory

`downloadOcrModels` built a three-entry file list: `_det.onnx`, `_cls.onnx`,
`_rec_<modelType>.onnx`. But `resolveModelPaths` resolves a *fourth* path,
`<ocrVersion>_rec_<modelType>_dict.txt`, and the `Engine` constructor falls back
to it whenever the recognizer ONNX carries no `character` metadata.

So "download programmatically" — a path we documented — could hand you a models
directory that fails at engine construction, with nothing in the download
result hinting at why. The downloader now fetches the dict as well. A 404 on it
is non-fatal, because some ONNX genuinely embed their charset, but it is
surfaced in the result rather than silently skipped. Details in
[Model downloader](../../api/downloader.md).

## The CLI is the wrappers' entire API

Three of the four out-of-tree wrappers — `arbo-ocr-go`, `arbo-ocr-rust`,
`arbo-ocr-php` — plus the `arbo-ocr-python` PyPI package drive `arboocr_demo`
as a subprocess and parse its `--json` stdout. For all of them the CLI is not a
convenience; it is the whole public API. And the CLI exposed a strict subset of
`EngineConfig`, which meant **none of the accuracy tuning from the last cycle
was reachable from Go, Rust, or PHP at all**.

The detection and recognition knobs are now flags: `--det-thresh`,
`--det-box-thresh`, `--det-unclip-ratio`, `--split-overmerged`,
`--min-confidence`, `--rec-batch-num`, `--clahe`, plus per-model path overrides.
Three more:

| Flag | What it does |
|---|---|
| `--log-level debug\|info\|warn\|error` | Engine events to stderr; silent by default, so `--json` stdout stays clean |
| `--draw <path>` | Writes a copy of the image with detected boxes outlined |
| `--word-boxes` | Emits a polygon per word (per character for CJK) |

Related, and in the same round: constructing the engine with a missing or
corrupt model used to throw `Ort::Exception` straight out of the `Engine`
constructor, past `main()`, into `std::terminate()`. The subprocess wrappers saw
a non-zero exit and stderr noise. It is caught now. Full flag reference on
[Command line](../../cli.md).

## Byte-buffer input

`cv::imdecode` appeared nowhere in the tree. `recognize()` took either a file
path or an already-decoded `cv::Mat` — useful from C++ and from the numpy path
in the Python bindings, useless from a web handler holding a PNG upload. Every
web integration was therefore writing a temp file per request.

`Engine::recognizeEncoded(const uint8_t* data, size_t size)` decodes in memory
and forwards to the same pipeline. It is deliberately a distinct name rather
than a `recognize()` overload, because a raw-pointer overload sitting next to
`recognize(const cv::Mat&)` binds to too many things by accident.

## A visualizer, boxes only

RapidOCR ships `.vis("out.jpg")` on every result object; it is in essentially
every usage example they publish. arboOCR's README told you to draw the boxes
yourself with `cv::polylines` — a suggestion, not an API.

`drawResult(const cv::Mat&, const PagePrediction&)` in `arboOCR/visualize.hpp`
returns an annotated copy. OpenCV was already a hard dependency, so this cost
nothing on the dependency surface. It draws **boxes only, no text**, and that is
a deliberate limitation rather than an unfinished feature: `cv::putText` cannot
render CJK, and a visualizer that silently mangles half the scripts the
recognizer supports is worse than one that does not try. See
[Visualization](../../api/visualize.md).

## Installable, finally

`arboOCR` was declared `STATIC` and that was the end of it — no `install()`, no
`export()`, no config package. There was no way to `find_package(arboOCR)`;
consumers had to vendor the entire source tree, which flatly contradicted the
README's "drop it into your own project" pitch.

There is now an install/export setup, so `find_package(arboOCR CONFIG)` works
against an installed tree. The awkward part was that the ONNXRuntime link path
is conditional on `ARBOOCR_USE_SYSTEM_DEPS`, so the exported config has to
handle both branches.

## Word and character boxes, with the caveat stated up front

The recognizer already decoded per-token with CTC timestep positions and
per-character scores — that is exactly how gap-to-space injection knows where to
insert spaces. Then the engine averaged the whole vector into one float and
threw the positions away.

Setting `EngineConfig::returnWordBoxes` (or passing `--word-boxes`) keeps them
and maps them back through the existing box geometry into page coordinates. You
get a `WordBox` per word, with its own polygon and its own mean CTC confidence.
Scripts that delimit words with spaces group into words; CJK characters stand
alone, because there is no space to split on. This mirrors RapidOCR's
`return_word_box`.

The honest caveat: these are derived from **CTC timestep alignment**, not from a
second detection pass. A timestep lands partway through a glyph, so the boxes
run roughly **half a character wide of the true position** — they lag right.
That makes them good for highlighting, word-level search, and column/reading
structure. It makes them **not** good for cropping individual glyphs out of the
page. If you need pixel-accurate glyph boxes, this is not that.

## What we deliberately did not adopt

Four backends that all look like wins, declined on RapidOCR's own measurements:

| Not adopted | Why, per RapidOCR's numbers |
|---|---|
| **CoreML EP** | 3.16x–14.04x *slower* than the CPU provider on a MacBook Pro M2. PP-OCRv5 rec mobile: 18.50 ms CPU vs 259.70 ms CoreML. Accuracy bit-identical, so there is nothing to trade for. |
| **OpenVINO** | Genuinely the fastest CPU engine they measured — PP-OCRv6 medium det 0.4476 s vs ONNXRuntime 0.9491 s at the same H-mean. But it does not release memory after large images ([openvino#11939](https://github.com/openvinotoolkit/openvino/issues/11939)), which is precisely the failure mode we just spent the arena change fixing. |
| **MNN** | Faster on detection (v4 det mobile 0.182 → 0.159 s), slower on recognition (v4 rec mobile 0.0176 → 0.0213 s; v5 rec mobile 0.0196 → 0.0373 s; v5 rec server 0.0582 → 0.0724 s). Identical accuracy throughout. A wash, in exchange for a second runtime dependency. |
| **ONNXRuntime CUDA EP** | Measured *slower* than CPU on a GTX 1660 Super (2.574 vs 1.183 s/img) **and** on an RTX 3090 (0.999 vs 0.505 s/img), because OCR detection is inherently dynamic-shaped. RapidOCR's conclusion is a flat "not recommended". |

Read the OpenVINO and CUDA rows together and the pattern is clear enough:
"faster on paper" and "faster on your workload" are different claims, and a GPU
does not automatically help a dynamic-shape model. arboOCR's `useCuda = false`
default was already right; what was missing was the reason, not the setting.

Staying single-runtime on ONNXRuntime remains the correct call. None of these
four buys enough to justify a second inference dependency, and two of them are
straightforwardly worse.

## Still open

`useFp16 = true` is the TensorRT default and **no test exercises it**. The
existing evidence covers less than it looks like it does: our Jetson
verification reports CPU and TensorRT producing identical text, but that was
`tiny` models on one receipt, and the library default is `small`.

RapidOCR separately documents an H-mean collapse with TensorRT FP16 on server
detection models, which is exactly the shape of failure an untested default
should not be sitting on. An A/B harness exists — `scripts/fp16_ab.py` in the
library repo — but it needs NVIDIA hardware and the dev machine is an AMD RX
6600. It runs on the Jetson on **2026-08-10**, at `small` and `medium`.

Until then, treat `useFp16` as an unvalidated default rather than a measured
one. That distinction is the whole point of this post.
