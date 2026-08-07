---
title: Visualization
---

# Visualization

One function, for the moment when the JSON looks plausible but you need to see
where the boxes actually landed.

```cpp
#include <arboOCR/visualize.hpp>

cv::Mat drawResult(const cv::Mat& image, const PagePrediction& page);
```

It returns a **copy** of `image` with every line's polygon outlined. The
caller's `Mat` is never modified and the result never shares its buffer, so
there is no aliasing surprise if you keep drawing on it or hand it to another
thread.

## Usage

```cpp
#include <arboOCR/engine.hpp>
#include <arboOCR/visualize.hpp>
#include <opencv2/imgcodecs.hpp>

arbo::ocr::EngineConfig cfg;
arbo::ocr::Engine engine(cfg);

const std::string path = "receipt.jpg";
auto page = engine.recognize(path);

cv::Mat src = cv::imread(path);
cv::imwrite("receipt.boxes.png", arbo::ocr::drawResult(src, page));
```

`drawResult` takes the image separately because `PagePrediction` carries only
the filename, not the pixels — the engine does not hold your image alive after
the call. If you recognized from a `cv::Mat` or from
[encoded bytes](index.md#three-ways-in-path-cvmat-encoded-bytes), pass the Mat
you already have (or `decodeImageBytes()` on the buffer); there is no path to
re-read.

## Colours mean something

| Colour | Condition |
|---|---|
| Green | Line recognition score **at or above** `0.5` |
| Red | Line recognition score **below** `0.5` |

`0.5` is not a new number invented for the overlay. It is
[`EngineConfig::minimumConfidence`](index.md#engineconfig-field-reference)'s
default — the bar the pipeline itself uses to drop lines.

!!! tip "A red box means you lowered the filter"
    With default config you will **never** see red: those lines were already
    discarded before they reached `page.lines`. Red only appears once you have
    lowered or disabled `minimumConfidence`, which is exactly when you are
    debugging "why did this line disappear". Green-vs-red then reads directly
    as "kept by default" vs "would have been dropped".

A 1-channel input is promoted to BGR so those two colours stay
distinguishable — on a grayscale `Mat` both would collapse into channel 0 and
the whole point would be lost. Any other input yields a `Mat` of the same size
and type.

## Never throws

Consistent with the rest of the library:

- An empty `image` returns an empty `Mat`.
- Empty `page.lines` returns a plain copy.
- Polygons with fewer than 2 points are skipped.
- Off-image coordinates are clipped by OpenCV's own drawing code.

## It does not render the recognized text

!!! warning "Boxes only — no text overlay, by design"
    `drawResult` draws polygons and nothing else. `cv::putText` cannot render
    CJK (the Hershey fonts OpenCV ships are a Latin subset), and arboOCR's
    PP-OCRv6 models are **multilingual by default** — so a text overlay would
    be silently wrong or blank for a large share of real input, which is worse
    than no overlay at all.

    Doing it properly requires `opencv_freetype` plus a bundled TTF with the
    coverage to match. RapidOCR solves this by downloading a font at runtime;
    that is not a dependency arboOCR takes.

If you need labelled output, build it in your own code: you already have
`line.text`, `line.score` and `line.polygon`, so draw on top of `drawResult`'s
return value with whatever text stack your application already links —
`opencv_freetype`, Pango, Skia, or an HTML/SVG overlay if the result is headed
for a browser anyway. Rendering in the browser is usually the least work: the
polygon coordinates are already in source-image space.

For per-word rather than per-line geometry, enable
[`returnWordBoxes`](index.md#word-boxes) and draw `line.words` yourself —
`drawResult` outlines lines only. Read the accuracy caveat on that page first;
word polygons are approximate.

## From the CLI

The demo CLI exposes this as `--draw <path>`, which writes the overlay next to
whatever else you asked for (it composes with `--json`). See the
[CLI reference](../cli.md).

## See also

- [API Reference](index.md#arboocrengine) — `Engine`, `PagePrediction`, and the JSON schema.
- [Logging](logging.md) — the other half of debugging a pipeline that returns nothing.
