---
title: PHP
---

# PHP

PHP wrapper for arboOCR. It runs the prebuilt `arboocr_demo` binary via
`proc_open` — **no C++ build required**, no PHP extension to compile.

- **Package:** `arbo/ocr-php` (Packagist)
- **Namespace:** `Arbo\Ocr`
- **Repo:** [ARBO-TEAM/ArboOcrPhp](https://github.com/ARBO-TEAM/ArboOcrPhp)
- **Config field style:** `camelCase`

## Install

```bash
composer require arbo/ocr-php
```

On install, a **Composer post-install hook** downloads the matching arboOCR
release binary (Windows or Linux, auto-detected) into `bin/<platform>/` inside
the installed package. PHP is the only wrapper that gets this for free —
Composer's `post-install-cmd` runs arbitrary code at install time, which pip,
Cargo, and Go modules have no equivalent of, so those three download lazily
instead.

The pinned release is
[`v0.3.0`](https://github.com/wafik/ArboOCR/releases/tag/v0.3.0). The
auto-download is live and verified working end to end — no manual binary step
needed.

!!! tip "If the download fails"
    On an offline install or an unsupported OS, download a release manually from
    the [arboOCR releases page](https://github.com/wafik/ArboOCR/releases) and
    pass `binPath` explicitly — see [Manual binary path](#manual-binary-path).
    This is also the pattern to use when your deploy runs
    `composer install --no-scripts`, which skips the hook.

## Models

`arbo/ocr-php` does not bundle OCR models — it does not have to. The
`arboocr_demo` binary it spawns downloads the stock weights it is missing on
first use, checks each file against a SHA-256 baked into the binary, and caches
it per platform. Point `modelsDir` at a folder that already holds the PP-OCRv6
ONNX files and nothing touches the network. For the default
`modelType: 'small'` that means three files:

```text
models/
├── PP-OCRv6_det.onnx
├── PP-OCRv6_rec_small.onnx
└── PP-OCRv6_rec_small_dict.txt
```

`PP-OCRv6_cls.onnx` is only needed when `useAngleCls` is on. Swap the `_small`
files for `_tiny` or `_medium` to change size — or keep all three in the same
directory and switch with `modelType`.

Full file matrix, the cache locations, and how to turn the download off:
[Models](index.md#models) on the wrappers overview, or
[Models](../models/index.md) for the complete treatment.

## Usage

```php
use Arbo\Ocr\Engine;

$engine = new Engine([
    'modelsDir' => '/path/to/models',
    // 'binPath' => '/custom/path/to/arboocr_demo', // optional override
    // 'modelType' => 'small', // tiny/small/medium — default small
    // 'useAngleCls' => true,
    // 'useCuda' => true,
]);

$result = $engine->recognize('/path/to/image.jpg');

echo $result->backend, "\n";       // cpu / cuda / tensorrt
foreach ($result->lines as $line) {
    echo $line->text, ' (', $line->score, ")\n";
}
```

## Configuration

`new Engine([...])` takes an options array:

| Key | Type | Default | Description |
|---|---|---|---|
| `modelsDir` | `string` | — | Directory holding the PP-OCRv6 ONNX files |
| `binPath` | `string` | Composer-installed path | Explicit path to `arboocr_demo` |
| `modelType` | `string` | `'small'` | Recognizer size: `'tiny'`, `'small'`, or `'medium'` |
| `useAngleCls` | `bool` | `false` | Enable angle classification (requires `PP-OCRv6_cls.onnx`) |
| `useCuda` | `bool` | `false` | Request the CUDA execution provider |

### Manual binary path

`binPath` is the manual override. Set it when the Composer hook did not run, or
when you ship the binary outside `vendor/` — for example, baked into a Docker
image layer so the runtime image never touches the network:

```php
$engine = new Engine([
    'modelsDir' => '/path/to/models',
    'binPath'   => '/opt/arboocr/arboocr_demo',
]);
```

## Error handling

`Engine::recognize()` throws `Arbo\Ocr\OcrException` only when the process
itself fails to start, exits non-zero, or produces unparseable output.

```php
use Arbo\Ocr\OcrException;

try {
    $result = $engine->recognize('/path/to/image.jpg');
} catch (OcrException $e) {
    // binary missing, non-zero exit, or unparseable JSON
    error_log($e->getMessage());
}
```

!!! note "Empty results are not errors"
    An empty `$result->lines` array means no text was found in the image. That
    is a successful call with zero detections, not an exception. Check
    `count($result->lines)`, not the `try`/`catch`, to distinguish "no text"
    from "broken pipeline".

## Result shape

`Engine::recognize()` returns a result object:

| Property | Description |
|---|---|
| `$result->backend` | Execution provider actually used: `cpu`, `cuda`, or `tensorrt` |
| `$result->lines` | Array of recognized lines, in detection order |
| `$result->elapsedMs` | Engine-reported inference time in milliseconds |

Each entry in `$result->lines` carries:

| Property | Description |
|---|---|
| `$line->text` | Recognized text for the line |
| `$line->score` | Recognition confidence, `0.0`–`1.0` |

## Quick example: tiny model

For a fast local smoke test, use `modelType: 'tiny'` — the smallest and fastest
PP-OCRv6 recognizer. A local arboOCR checkout's `models/` folder already
contains the tiny det/rec/cls ONNX files, so no extra download is needed:

```php
use Arbo\Ocr\Engine;

$engine = new Engine([
    'modelsDir' => '/path/to/arboOCR/models', // e.g. a local arboOCR checkout's models/ dir
    'modelType' => 'tiny',
]);

$result = $engine->recognize('/path/to/receipt.jpg');

printf("backend=%s lines=%d elapsedMs=%.1f\n", $result->backend, count($result->lines), $result->elapsedMs);
foreach ($result->lines as $line) {
    printf("  %-40s score=%.3f\n", $line->text, $line->score);
}
```

`tiny` trades accuracy for speed — good for quick local testing. Switch to
`small` (the default) or `medium` for production-quality recognition.

## How it works

The package never builds or vendors arboOCR's C++ source. It downloads a
prebuilt, self-contained bundle from arboOCR's GitHub Releases — the binary plus
its required shared libraries, no source and no ONNX models — and calls it as a
subprocess per image with a `--json` flag, parsing the JSON result.

The full design is written up in arboOCR's
[PHP integration design spec](https://github.com/wafik/ArboOCR/blob/main/docs/superpowers/specs/2026-07-27-php-integration-design.md).

!!! warning "Two subprocess pitfalls this wrapper works around"
    `arboocr_demo` can write ~200 KB of ONNXRuntime warnings to stderr before
    any stdout appears — enough to deadlock a naive sequential pipe read, so the
    wrapper drains both streams rather than reading them in sequence.
    Separately, boolean flags must be passed as a single `--flag=value` token:
    `arboocr_demo`'s argument parser (cxxopts) only binds a bool's value via
    `=`, and the two-token form leaves the flag implicitly `true` regardless of
    intent. Both were fixed in this wrapper after the second one crashed
    `arboocr_demo` in production.

## Performance

Wrapper overhead only — subprocess spawn minus the engine's own reported
inference time. Accuracy is identical across all wrappers because they call the
same binary.

| Model size | PHP | Go | Rust | JavaScript | Python |
|---|---:|---:|---:|---:|---:|
| `tiny` | 196 ms | 174 ms | 135 ms | 186 ms | 218 ms |
| `small` | 217 ms | 177 ms | 162 ms | 216 ms | 246 ms |
| `medium` | 291 ms | 235 ms | 233 ms | 282 ms | 317 ms |

PHP runs roughly 55–65 ms above Go and Rust: `php.exe` interpreter startup on
top of `proc_open`, versus a compiled binary paying only process-spawn cost.
Python lands in the same ballpark as PHP for the same reason.

!!! tip "Amortising the interpreter cost"
    Under PHP-FPM or a long-lived worker the interpreter is already warm, so the
    real per-call cost is closer to the bare `proc_open` spawn. The measured gap
    above reflects cold CLI invocations.

## License

Apache-2.0.

## See also

- [Language wrappers](index.md) — shared architecture and the trade-off
- [Models](../models/index.md) — obtaining PP-OCRv6 ONNX files
- [Quickstart](../quickstart.md) — CLI and C++ entry points
