---
title: Models
---

# Models

arboOCR ships no weights. It needs PP-OCRv6 ONNX files in `modelsDir`:

```text
models/
├── PP-OCRv6_det.onnx                text detection (single file — no size variants)
├── PP-OCRv6_cls.onnx                angle classification (only needed if useAngleCls)
├── PP-OCRv6_rec_medium.onnx         text recognition — tiny | small | medium
└── PP-OCRv6_rec_medium_dict.txt     character dict (only if not embedded in ONNX metadata)
```

## Two ways to get them

??? note "Option A — copy from an existing install"

    If you already have a Python `rapidocr` install, copy its `models/`
    directory into `modelsDir`, renaming files to match the layout above.

??? note "Option B — download programmatically"

    ```cpp
    arbo::ocr::downloadOcrModels(
        "https://your-host.example/models/PP-OCRv6/", // baseUrl — you supply this
        "PP-OCRv6", "medium", "models");
    ```

    See [Model downloader](../api/downloader.md) for the full signature and
    return type.

!!! warning "arboOCR ships **no default download URL**"

    This is deliberate, not an omission. PP-OCR model hosting locations aren't
    stable across mirrors, so the caller picks the source — you pass `baseUrl`
    and you own the decision about where your weights come from. There is no
    silent fallback host that can rot, move, or serve you a different model
    than the one you tested against.

## Which files do I actually need?

Not all four files are mandatory, and the ones that vary by size only vary for
the recognizer.

| File | When you need it | Varies by `modelType`? |
|---|---|---|
| `PP-OCRv6_det.onnx` | **Always.** Detection runs on every page. | No — one detector file, no size variants |
| `PP-OCRv6_rec_<size>.onnx` | **Always**, one per size you actually use | Yes — `tiny` / `small` / `medium` |
| `PP-OCRv6_rec_<size>_dict.txt` | Per size, only if the character list isn't embedded in the ONNX metadata | Yes — pairs with its recognizer |
| `PP-OCRv6_cls.onnx` | Only if `useAngleCls` is on | No |

You only need the recognizer size(s) you will actually use — there is no
requirement to have all three on disk. But `modelsDir` can hold all three
side by side, so a benchmarking or A/B setup can keep `tiny`, `small`, and
`medium` together and switch between them by changing `modelType` alone, with
no file shuffling.

!!! tip "Custom or fine-tuned weights"

    The flat `<ocrVersion>_*` names above are just defaults. Point
    `recModelPath` / `dictPath` / `detModelPath` / `clsModelPath` at arbitrary
    files to override any of them — see [Languages](languages.md).

## In this section

| Page | What it answers |
|---|---|
| [Choosing a model size](sizes.md) | `tiny` vs `small` vs `medium`, with measured latency and accuracy |
| [Languages](languages.md) | Why there is no `language` config, and how to load custom models |
| [Low-contrast documents (CLAHE)](clahe.md) | Rescuing faded scans where the detector misses boxes entirely |
| [Accuracy defaults](accuracy-defaults.md) | **Read this if you are upgrading.** Defaults changed; several are breaking |

New here? Start at the [Quickstart](../quickstart.md) instead — it needs these
files, but it will tell you the minimum to get one image through.
