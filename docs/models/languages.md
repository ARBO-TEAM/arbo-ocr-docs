---
title: Languages
---

# Languages (PP-OCRv6)

Default PP-OCRv6 recognition models are **already multi-language** in a single
ONNX file (PaddleOCR: medium/small cover 50 languages including Chinese,
English, Japanese, and 46 Latin-script languages; tiny is similar without
Japanese). Load the default models and you get that coverage.

| modelType | Language coverage |
|---|---|
| `medium` / `small` | 50 languages — Chinese, English, Japanese, and 46 Latin-script languages |
| `tiny` | Similar, **without** Japanese |

!!! info "There is no `language` config — by design"

    arboOCR has **no `language` field** in `EngineConfig`. That is not a missing
    feature: the recognizer weights are already multi-language, so a language
    selector would either be a no-op or an invitation to pick the wrong model.
    You load the default models and you get the coverage above. If you need a
    script the model wasn't trained on, you don't flip a flag — you supply
    different weights.

## Custom and fine-tuned models

For scripts outside the model's training set, or a fine-tuned ONNX, point
explicit paths at your files (empty = default names under `modelsDir`):

```cpp
arbo::ocr::EngineConfig cfg;
cfg.modelsDir = "models";
cfg.recModelPath = "models/custom_rec.onnx";
cfg.dictPath = "models/custom_dict.txt"; // only if ONNX metadata has no character list
// cfg.detModelPath / cfg.clsModelPath likewise if needed
```

Note the `dictPath` comment: a dict file is only required when the ONNX
metadata doesn't carry the character list. Ship the dict with your fine-tuned
model if you aren't sure.

## Helpers

`resolveModelPaths(cfg)` returns the four resolved paths — det, cls, rec, dict
— after applying your overrides on top of the `modelsDir` defaults. It does
**not** check that those files exist; it is a pure path computation, so use it
to log or validate what the engine is *about* to open, not as an existence
check.

For custom downloads, use `downloadFile(url, dest)` to fetch straight into
those paths. `downloadOcrModels` still writes the default flat names only — it
is not a general-purpose fetcher for arbitrarily-named weights.

See [Model downloader](../api/downloader.md) for both functions, and
[Models](index.md) for which files you actually need on disk.
