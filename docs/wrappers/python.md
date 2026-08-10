---
title: Python
---

# Python

Python wrapper for arboOCR. It runs the prebuilt `arboocr_demo` binary via
`subprocess` — **no C++ build required**.

- **Package:** `arbo-ocr-python` (PyPI)
- **Import name:** `arbo_ocr`
- **Repo:** [ARBO-TEAM/arbo-ocr-python](https://github.com/ARBO-TEAM/arbo-ocr-python)
- **Config field style:** `snake_case`

!!! note "Two different Python paths into arboOCR"
    This page covers the **subprocess wrapper** — it spawns `arboocr_demo` once
    per image and parses its JSON. That is not the same thing as the in-process
    **pybind11 bindings** described in
    [Quickstart](../quickstart.md#python-bindings-optional), which load the C++
    engine into your interpreter, accept NumPy arrays directly, and have no
    per-call process spawn — at the cost of needing a built arboOCR. Use this
    package when you want a one-line install; use the bindings when you want
    throughput.

## Install

```bash
pip install arbo-ocr-python
arbo-ocr-install
```

### Why the second command exists

Composer can run arbitrary code after installing a package via its
`post-install-cmd` hook. **pip cannot** — installing a wheel does not reliably
execute package code, so there is no safe place to hang a binary download off
`pip install`.

So this package ships an explicit console script instead. `arbo-ocr-install` is
run once after installing, exactly the same pattern as `playwright install`. It
detects the platform (Windows or Linux), downloads the matching arboOCR release
binary, and unpacks it into `arbo_ocr/bin/<platform>/`.

The pinned release is
[`v0.3.0`](https://github.com/wafik/ArboOCR/releases/tag/v0.3.0).

!!! tip "If the download fails"
    On an offline install or an unsupported OS, grab a release manually from
    the [arboOCR releases page](https://github.com/wafik/ArboOCR/releases) and
    pass `bin_path` explicitly — see [Manual binary path](#manual-binary-path).

## Models

`arbo-ocr-python` does not bundle OCR models — it does not have to. The
`arboocr_demo` binary it spawns downloads the stock weights it is missing on
first use, checks each file against a SHA-256 baked into the binary, and caches
it per platform. Point `models_dir` at a folder that already holds the PP-OCRv6
ONNX files and nothing touches the network. For the default
`model_type="small"` that means three files:

```text
models/
├── PP-OCRv6_det.onnx
├── PP-OCRv6_rec_small.onnx
└── PP-OCRv6_rec_small_dict.txt
```

`PP-OCRv6_cls.onnx` is only needed when angle classification is on. Swap the
`_small` files for `_tiny` or `_medium` to change size — or keep all three in
the same directory and switch with `model_type`.

Full file matrix, the cache locations, and how to turn the download off:
[Models](index.md#models) on the wrappers overview, or
[Models](../models/index.md) for the complete treatment.

## Usage

```python
from arbo_ocr import Engine

engine = Engine(models_dir="/path/to/models", model_type="small")
result = engine.recognize("/path/to/image.png")

for line in result.lines:
    print(line.text, line.score)
```

### Manual binary path

Pass `bin_path` to point at a manually-downloaded binary instead of relying on
the one `arbo-ocr-install` placed:

```python
engine = Engine(bin_path="/custom/path/to/arboocr_demo", models_dir="/path/to/models")
```

## Configuration

| Parameter | Type | Default | Description |
|---|---|---|---|
| `models_dir` | `str` | — | Directory holding the PP-OCRv6 ONNX files |
| `model_type` | `str` | `"small"` | Recognizer size: `"tiny"`, `"small"`, or `"medium"` |
| `bin_path` | `str` | Auto-installed path | Explicit path to `arboocr_demo`, bypassing auto-install |

!!! warning "Angle classification and CUDA"
    The `arbo-ocr-python` README documents only the three parameters above. The
    Go, Rust, and PHP wrappers additionally document angle-classification and
    CUDA toggles (`use_angle_cls` / `use_cuda` and their per-language spellings).
    If you need those from Python, check the installed package's own
    `Engine.__init__` signature before relying on them — see
    [Go](go.md#configuration), [Rust](rust.md#configuration), and
    [PHP](php.md#configuration) for the equivalents.

## Result shape

`Engine.recognize()` returns a result object with these attributes:

| Attribute | Description |
|---|---|
| `backend` | Execution provider actually used: `cpu`, `cuda`, or `tensorrt` |
| `lines` | List of recognized lines, in detection order |
| `elapsed_ms` | Engine-reported inference time in milliseconds |

Each entry in `lines` carries:

| Attribute | Description |
|---|---|
| `text` | Recognized text for the line |
| `score` | Recognition confidence, `0.0`–`1.0` |

## Quick example: tiny model

For a fast local smoke test, use `model_type="tiny"` — the smallest and fastest
PP-OCRv6 recognizer. A local arboOCR checkout's `models/` folder already
contains the tiny det/rec/cls ONNX files, so no extra download is needed:

```python
from arbo_ocr import Engine

engine = Engine(
    models_dir="/path/to/arboOCR/models",  # e.g. a local arboOCR checkout's models/ dir
    model_type="tiny",
)

result = engine.recognize("/path/to/receipt.jpg")

print(f"backend={result.backend} lines={len(result.lines)} elapsed_ms={result.elapsed_ms:.1f}")
for line in result.lines:
    print(f"  {line.text:<40} score={line.score:.3f}")
```

`tiny` trades accuracy for speed — good for quick local testing. Switch to
`small` (the default) or `medium` for production-quality recognition.

## How it works

The package never builds or vendors arboOCR's C++ source. `arbo-ocr-install`
downloads a prebuilt, self-contained bundle from arboOCR's GitHub Releases —
the binary plus its required shared libraries, no source and no ONNX models.
`Engine.recognize()` then invokes that binary as a subprocess per image with a
`--json` flag and parses the JSON result.

Two implementation details worth knowing, both already fixed upstream:

- **UTF-8 decoding is explicit.** `subprocess.run(text=True)` without an
  explicit `encoding` decodes using the Windows locale default (`cp1252`), not
  UTF-8. Recognized text containing a byte `cp1252` cannot map raises
  `UnicodeDecodeError` inside `subprocess`'s internal reader thread, where it is
  silently swallowed — surfacing only as `stdout`/`stderr` being `None` despite
  `returncode == 0`. `Engine.recognize()` now passes `encoding="utf-8"`
  explicitly.
- **Import cost was trimmed.** `installer.py` used to import `urllib.request`,
  `zipfile`, and `tarfile` at module level even though `engine.py` only needs
  `detect_platform()` / `default_bin_path()`. Those are heavy stdlib imports
  (`urllib.request` alone pulls in `ssl` and `http.client`), costing ~50 ms of
  interpreter startup per call for nothing. Moving them into the functions that
  actually use them cut measured overhead by ~35–40 ms per call at every model
  size.

## Performance

Wrapper overhead only — subprocess spawn minus the engine's own reported
inference time, over a 40-image SROIE sample. Accuracy is identical across all
four wrappers because they call the same binary.

| Model size | Python | Go | Rust | PHP |
|---|---:|---:|---:|---:|
| `tiny` | 203 ms | 138 ms | 131 ms | 192 ms |
| `small` | 249 ms | 184 ms | 174 ms | 234 ms |
| `medium` | 321 ms | 245 ms | 253 ms | 302 ms |

Python sits alongside PHP: both pay their interpreter's startup cost on top of
the same `arboocr_demo` spawn that Go and Rust pay alone. If that floor is
unacceptable, use the in-process
[pybind11 bindings](../quickstart.md#python-bindings-optional) instead.

## Development

```bash
pip install -e ".[dev]"
pytest
```

## License

Apache-2.0.

## See also

- [Language wrappers](index.md) — shared architecture and the trade-off
- [Models](../models/index.md) — obtaining PP-OCRv6 ONNX files
- [Quickstart](../quickstart.md) — CLI, C++, and in-process Python bindings
