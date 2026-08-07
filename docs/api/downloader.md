---
title: Model downloader
---

# Model downloader

arboOCR does not bundle model files. If you would rather fetch them at
deploy time than ship them in your image, `model_downloader.hpp` gives you
two functions:

```cpp
DownloadResult downloadFile(const std::string& url, const std::string& destPath);
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

It writes **only the default flat filenames** — the same layout the
`EngineConfig` path derivation expects when `detModelPath`, `clsModelPath`,
`recModelPath` and `dictPath` are left empty. Given `ocrVersion`, `modelType`
and `modelsDir`, it fetches the set that pairs with them and drops it into
`modelsDir` with no subdirectories and no renaming hooks.

!!! tip "Custom or fine-tuned models: use `downloadFile`"
    If your weights do not follow the default naming — a fine-tuned
    recognizer, a language-specific dictionary, an A/B variant you keep beside
    the stock file — call `downloadFile` once per artefact into explicit
    destination paths, then point the matching `EngineConfig::*ModelPath` /
    `dictPath` field at each one. `downloadOcrModels` has no escape hatch for
    that; it is a convenience for the stock layout only.

Both functions return `DownloadResult`. Check it. A partial download leaves
you with a file that exists and does not load, and by the time `Engine` sees
it you get an empty-lines page rather than an exception — see the
[API Reference](index.md#arboocrengine).

## See also

- [Models](../models/index.md) — the expected layout, and how to get the files without downloading.
- [Languages](../models/languages.md) — which dictionary pairs with which recognizer.
