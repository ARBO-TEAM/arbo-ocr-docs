---
title: Models
---

# Models

arboOCR ships no weights — the library binary contains no model data. It needs
PP-OCRv6 ONNX files, and as of `models-v1` it will fetch the stock ones for you
if it cannot find them:

```text
models/
├── PP-OCRv6_det.onnx                text detection (single file — no size variants)
├── PP-OCRv6_cls.onnx                angle classification (only needed if useAngleCls)
├── PP-OCRv6_rec_medium.onnx         text recognition — tiny | small | medium
└── PP-OCRv6_rec_medium_dict.txt     character dict (only if not embedded in ONNX metadata)
```

## Three ways to get them

??? note "Option A — nothing (the default)"

    Construct an `Engine` with an empty or incomplete `modelsDir` and arboOCR
    downloads what is missing, verifies it against a SHA-256 compiled into the
    binary, and loads it. There is no flag to set and no first-run step to
    document for your users.

    Files land in a per-platform cache, not in your build tree, so several
    projects on one machine share a single copy:

    | Platform | Path |
    |---|---|
    | Windows | `%LOCALAPPDATA%\arboOCR\models\models-v1` |
    | macOS | `~/Library/Caches/arboOCR/models/models-v1` |
    | Linux | `$XDG_CACHE_HOME/arboOCR/models/models-v1`, else `~/.cache/arboOCR/models/models-v1` |

    The trailing `models-v1` is the release tag the weights are pinned to. It
    is part of the path on purpose: a future `models-v2` gets its own
    directory rather than reusing a same-named file from this one.

    Set `autoDownload = false`, or `ARBOOCR_OFFLINE=1`, if you would rather it
    never reached the network. See
    [Model downloader](../api/downloader.md#turning-it-off).

??? note "Option B — copy from an existing install"

    If you already have a Python `rapidocr` install, copy its `models/`
    directory into `modelsDir`, renaming files to match the layout above.

    A populated `modelsDir` wins over any download — resolution checks it
    first, per file, so this costs zero network access even with
    `autoDownload` left on.

??? note "Option C — download programmatically"

    ```cpp
    arbo::ocr::downloadOcrModels(
        "",  // empty baseUrl — the default, pinned source
        "PP-OCRv6", "medium", "models");
    ```

    Pass your own `baseUrl` — an internal mirror, an artifact store, a CDN
    bucket you control — to fetch from somewhere else instead. Stock filenames
    are still hash-checked when they come from your host, so a mirror that has
    drifted is caught rather than loaded.

    Either way this fetches **all four** files above — detector, classifier,
    recognizer, and the `_dict.txt`. The dict is best-effort: a 404 on it is
    not fatal, because some ONNX genuinely carry their charset in `character`
    metadata and have no dict file to fetch. See
    [Model downloader](../api/downloader.md) for the full signature, the return
    type, and how to tell a best-effort miss from a real failure.

!!! info "There is a default download URL now — this page used to say there never would be"

    That earlier position was not arbitrary, and it is worth saying why it
    changed rather than quietly editing it out. The objection was three
    specific things: a hardcoded URL **rots**, you do not control **integrity**,
    and you do not control the **licence question** of where the weights came
    from. A default is only safe once all three have answers — and now they do.

    | Then | Now |
    |---|---|
    | The URL would rot | It points at an immutable release tag, `models-v1`. That is a version, not a moving target, and the tag is baked into the cache path so versions cannot collide. |
    | Integrity was yours to guarantee | Every stock file is checked against a SHA-256 compiled into the binary. A host serving different bytes is rejected, not loaded. |
    | Provenance was unstated | The weights live in [arbo-ocr-models](https://github.com/ARBO-TEAM/arbo-ocr-models) with a NOTICE recording their PP-OCR provenance; upstream PaddleOCR is Apache-2.0. |

    What has **not** changed is who decides, and the precedence still puts you
    first. An explicit `recModelPath` / `detModelPath` / `clsModelPath` /
    `dictPath` is used exactly as given and is never quietly replaced by a
    stock download. A file already sitting in `modelsDir` is used without
    touching the network. Your own `baseUrl` still overrides the source. The
    default is what happens when you express no preference — it does not
    overrule one.

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

    An explicit path is used exactly as written. If the file is missing you get
    a load failure, **not** a stock model downloaded in its place — a
    fine-tuned recognizer swapped for the stock one would still produce
    plausible-looking text, which is the kind of bug that survives a code
    review and a month of production.

## In this section

| Page | What it answers |
|---|---|
| [Choosing a model size](sizes.md) | `tiny` vs `small` vs `medium`, with measured latency and accuracy |
| [Languages](languages.md) | Why there is no `language` config, and how to load custom models |
| [Low-contrast documents (CLAHE)](clahe.md) | Rescuing faded scans where the detector misses boxes entirely |
| [Accuracy defaults](accuracy-defaults.md) | **Read this if you are upgrading.** Defaults changed; several are breaking |

New here? Start at the [Quickstart](../quickstart.md) instead — it needs these
files, but it will tell you the minimum to get one image through.
