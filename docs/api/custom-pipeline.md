---
title: Building a custom pipeline
---

# Building a custom pipeline

Need more control than `Engine` gives you? The three stages are public:

```cpp
arbo::ocr::Detector detector;
detector.loadModel("models/PP-OCRv6_det.onnx", /*useCuda=*/false, /*useTensorrt=*/true);
auto boxes = detector.getTextBoxes(image, scale, boxScoreThresh, boxThresh, unclipRatio);

arbo::ocr::Recognizer recognizer;
recognizer.loadModel("models/PP-OCRv6_rec_medium.onnx");
recognizer.loadKeysFromModelMetadata(); // or loadKeysFromFile("dict.txt")
auto lines = recognizer.getTextLines(croppedImages); // batched internally
```

`Classifier` (0°/180° orientation) is public too and slots between the two
when you need it — the `Engine` only runs it when `useAngleCls` is true.

## When to reach for this

The `Engine` facade is the right default. It resolves model paths, runs
detection → optional orientation → recognition, sorts lines into reading
order, applies `minimumConfidence`, and never throws. Dropping to the stages
means you own all of that.

Do it when:

- **You need the intermediate artefacts.** Boxes without text (layout
  analysis, table structure, redaction), or crops you want to inspect, cache,
  or feed somewhere else.
- **Detection and recognition run at different rates.** Detect once on a
  static form template, then recognize many filled copies against the same
  box set.
- **You are injecting your own stage.** Deskew, dewarp, per-region
  binarization, or a custom crop policy between detection and recognition.
- **You need a different fan-out.** One detector feeding a pool of
  recognizers, or per-region models (a numeric-only recognizer on the amount
  column, a general one elsewhere).
- **Recognition only.** You already have boxes from another source and want
  `getTextLines()` on your own crops.

!!! warning "You inherit the sharp edges"
    Reading order, confidence filtering, and the empty-result-instead-of-throw
    behaviour all live in the `Engine`. At stage level, `page.lines`-style
    top→bottom / left→right sorting does not happen for free, and errors
    surface however the stage surfaces them. Re-read
    [Accuracy defaults](../models/accuracy-defaults.md) before you
    re-implement the pipeline — those defaults are the difference between
    ~84% and ~94% char similarity on the receipt smoke set.

## Read the headers, not just this page

[`include/arboOCR/`](https://github.com/wafik/ArboOCR/tree/main/include/arboOCR)
carries full doc comments on each class. Three design decisions in particular
are documented inline where the code lives, because they are the ones people
otherwise "fix" and regress:

- **Why batching trades off the way it does.** `getTextLines()` batches
  internally, and that is a win on GPU and a loss on CPU — padding every crop
  to a shared width is wasted computation where there is no parallelism across
  the batch dimension. The numbers are in
  [Benchmarks](../benchmarks.md#batching-cpu-vs-tensorrt).
- **Why padding is normalized-then-padded, not padded-then-normalized.**
  Order matters here; getting it backwards changes what the network sees in
  the padded region.
- **Why the detector has no size variants.** `modelType` selects the
  recognizer only — there is a single `PP-OCRv6_det.onnx`.

## See also

- [Architecture](../architecture.md) — how the stages fit together in the facade.
- [Benchmarks](../benchmarks.md#batching-cpu-vs-tensorrt) — batching, backends, latency.
- [API Reference](index.md#arboocrengine) — the `Engine` path you are stepping around.
