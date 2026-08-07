---
title: Model downloader
---

# Model downloader

arboOCR does not bundle model files. If you would rather fetch them at
deploy time than ship them in your image, `model_downloader.hpp` gives you
three functions:

```cpp
struct DownloadResult {
    bool ok = false;
    std::string errorMessage;
    size_t bytesWritten = 0;
};

DownloadResult downloadFile(const std::string& url, const std::string& destPath);

std::vector<std::string> ocrModelFileNames(
    const std::string& ocrVersion,
    const std::string& modelType);

std::vector<DownloadResult> downloadOcrModels(
    const std::string& baseUrl, const std::string& ocrVersion,
    const std::string& modelType, const std::string& modelsDir);
```

Typical use:

```cpp
arbo::ocr::downloadOcrModels(
    "https://your-host.example/models/PP-OCRv6/", // baseUrl — you supply this
    "PP-OCRv6", "medium", "models");
```

!!! danger "There is no default URL, and there never will be"
    arboOCR ships **no default download URL** — PP-OCR model hosting
    locations aren't stable across mirrors, so the caller picks the source.
    A hardcoded default would be a link that rots, silently, in every
    deployment that ever pinned this version. You supply `baseUrl`: your own
    artifact store, an internal mirror, a CDN bucket you control. That also
    means you control integrity, availability, and the license question of
    where those weights came from.

## What `downloadOcrModels` actually writes

**Four files** — the three ONNX models *and* the recognizer dictionary. They
are exactly the default flat filenames the `EngineConfig` path derivation
expects when `detModelPath`, `clsModelPath`, `recModelPath` and `dictPath` are
left empty, dropped into `modelsDir` with no subdirectories and no renaming
hooks.

`ocrModelFileNames(ocrVersion, modelType)` returns that list, and both the URL
(`baseUrl + name`) and the destination (`modelsDir/name`) are derived from it.
Call it when you want to check what will land on disk, pre-seed a cache, or
write a manifest, without performing a download:

```cpp
for (const auto& name : arbo::ocr::ocrModelFileNames("PP-OCRv6", "medium")) {
    std::cout << name << "\n";
}
// PP-OCRv6_det.onnx
// PP-OCRv6_cls.onnx
// PP-OCRv6_rec_medium.onnx
// PP-OCRv6_rec_medium_dict.txt
```

`downloadOcrModels` returns **one `DownloadResult` per name, always four
entries, always in that same order** — index 0 detector, 1 classifier, 2
recognizer, 3 dictionary. The vector length does not vary with success or
failure, so you can index it positionally.

!!! warning "The dictionary is best-effort — index 3 may be `ok=false` and still fine"
    A recognizer that embeds its charset in the ONNX `character` metadata key
    needs no dict file at all, and repos hosting such models often do not
    publish one. A 404 there is therefore **not** promoted to an error for the
    whole call: the fourth `DownloadResult` keeps the normal shape, comes back
    `ok=false`, and has its `errorMessage` suffixed to say the failure may be
    ignorable — that the dict is only needed when the rec model does not embed
    its charset in the ONNX `character` metadata key.

    You are not lied to (`ok` is still `false`) and you are not forced to guess
    what it means. Treat the first three failing as fatal; treat the fourth as
    a prompt to check whether your recognizer carries its own charset. If it
    does not, and the dict did not download, `Engine` will produce
    empty-lines pages rather than throwing.

!!! tip "Custom or fine-tuned models: use `downloadFile`"
    If your weights do not follow the default naming — a fine-tuned
    recognizer, a language-specific dictionary, an A/B variant you keep beside
    the stock file — call `downloadFile` once per artefact into explicit
    destination paths, then point the matching `EngineConfig::*ModelPath` /
    `dictPath` field at each one. `downloadOcrModels` has no escape hatch for
    that; it is a convenience for the stock layout only.

## Checking results

Both download functions return `DownloadResult`. Check it. A partial download
leaves you with a file that exists and does not load, and by the time `Engine`
sees it you get an empty-lines page rather than an exception — see the
[API Reference](index.md#arboocrengine).

Neither function throws: an unreachable host or an HTTP error comes back as
`ok=false` with a message. `downloadFile` **skips** a destination that already
exists and is non-empty, returning `ok=true` with `bytesWritten = 0` — so
re-running a provisioning step is cheap and idempotent, and `bytesWritten == 0`
means "already had it", not "wrote nothing useful".

## See also

- [Models](../models/index.md) — the expected layout, and how to get the files without downloading.
- [Languages](../models/languages.md) — which dictionary pairs with which recognizer.
