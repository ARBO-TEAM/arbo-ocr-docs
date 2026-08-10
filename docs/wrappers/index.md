---
title: Language wrappers
---

# Language wrappers

arboOCR is a C++ library, but you do not need a C++ toolchain to use it. Five
official wrappers — [Python](python.md), [Go](go.md), [Rust](rust.md),
[PHP](php.md), and [JavaScript](js.md) — install with a single package-manager
command and talk to the same prebuilt engine.

## How all five work

Every wrapper is a thin process driver. None of them build, vendor, or link
arboOCR's C++ source. They download a prebuilt, self-contained release
bundle — the `arboocr_demo` binary plus its required shared libraries, no
source and no ONNX models — and run it as a subprocess per image with a
`--json` flag, parsing the JSON that comes back on stdout.

```text
your code
    │  Engine(...).recognize("page.jpg")
    ▼
wrapper ──── spawn ────► arboocr_demo --image page.jpg --models-dir models --json
    ▲                              │
    └────── JSON on stdout ────────┘
```

The compiled binary and its DLLs are language-agnostic: all five wrappers pull
the *same* release asset (`arboocr-windows-x64.zip` on Windows,
`arboocr-linux-x64.tar.gz` on Linux) and differ only in how they spawn it —
`subprocess` in Python, `os/exec` in Go, `std::process::Command` in Rust,
`proc_open` in PHP, `child_process` in JavaScript.

### The trade-off, stated honestly

!!! warning "One process per image"
    You pay a process spawn on every `recognize()` call. Measured wrapper
    overhead is roughly **130–320 ms per image** on top of the engine's own
    inference time (see [Overhead](#overhead) below). There is no in-process
    handle, no warm model cache you control, no async API, and no way to batch
    images into a single engine instance.

What you get in exchange: no compiler, no vcpkg, no CMake, no ABI to match, and
an install that is one line long. That is the right trade for scripting, batch
jobs, queue workers, and web backends where a request already costs tens of
milliseconds.

If the per-call overhead matters — high-throughput services, real-time loops,
anything that wants one loaded model serving many images — use the C++ library
directly (see the [API reference](../api/index.md)) or the in-process
[pybind11 bindings](../quickstart.md#python-bindings-optional), which keep the
engine alive in your process and skip the spawn entirely.

## Choosing a wrapper

| Language | Package / install | Repo | Config field style |
|---|---|---|---|
| [Python](python.md) | `pip install arbo-ocr-python` then `arbo-ocr-install` | [ARBO-TEAM/arbo-ocr-python](https://github.com/ARBO-TEAM/arbo-ocr-python) | `snake_case` |
| [Go](go.md) | `go get github.com/ARBO-TEAM/arbo-ocr-go` | [ARBO-TEAM/arbo-ocr-go](https://github.com/ARBO-TEAM/arbo-ocr-go) | `PascalCase` |
| [Rust](rust.md) | git dependency on `arbo-ocr` | [ARBO-TEAM/arbo-ocr-rust](https://github.com/ARBO-TEAM/arbo-ocr-rust) | `snake_case` |
| [PHP](php.md) | `composer require arbo/ocr-php` | [ARBO-TEAM/ArboOcrPhp](https://github.com/ARBO-TEAM/ArboOcrPhp) | `camelCase` |
| [JavaScript](js.md) | `npm install github:ARBO-TEAM/arbo-ocr-js` | [ARBO-TEAM/arbo-ocr-js](https://github.com/ARBO-TEAM/arbo-ocr-js) | `camelCase` |

The API surface is deliberately identical across all five — construct an engine
with a models directory, call `recognize` with an image path, read `backend`,
`lines`, and `elapsed_ms` off the result. Only the naming convention changes.
JavaScript is the one exception to the shape, and only because the language
forces it: `recognize` returns a `Promise`.

=== "Python"

    ```python
    from arbo_ocr import Engine

    engine = Engine(models_dir="/path/to/models", model_type="small")
    result = engine.recognize("/path/to/image.png")

    for line in result.lines:
        print(line.text, line.score)
    ```

=== "Go"

    ```go
    engine, err := arboocr.NewEngine(arboocr.Config{
        ModelsDir: "/path/to/models",
        ModelType: "small",
    })
    if err != nil {
        log.Fatal(err)
    }

    result, err := engine.Recognize("/path/to/image.jpg")
    if err != nil {
        log.Fatal(err)
    }

    for _, line := range result.Lines {
        fmt.Printf("%s (%.3f)\n", line.Text, line.Score)
    }
    ```

=== "Rust"

    ```rust
    use arbo_ocr::{Config, Engine};

    let engine = Engine::new(Config {
        models_dir: Some("/path/to/models".to_string()),
        model_type: Some("small".to_string()),
        ..Default::default()
    })?;

    let result = engine.recognize("/path/to/image.jpg")?;

    for line in &result.lines {
        println!("{} ({:.3})", line.text, line.score);
    }
    ```

=== "PHP"

    ```php
    use Arbo\Ocr\Engine;

    $engine = new Engine([
        'modelsDir' => '/path/to/models',
        'modelType' => 'small',
    ]);

    $result = $engine->recognize('/path/to/image.jpg');

    foreach ($result->lines as $line) {
        echo $line->text, ' (', $line->score, ")\n";
    }
    ```

=== "JavaScript"

    ```ts
    import { Engine } from "arbo-ocr-js";

    const engine = new Engine({
      modelsDir: "/path/to/models",
      modelType: "small",
    });

    const result = await engine.recognize("/path/to/image.jpg");

    for (const line of result.lines) {
      console.log(line.text, line.score.toFixed(3));
    }
    ```

## Getting the binary

Every wrapper auto-downloads the matching release asset for the current
platform (Windows or Linux, auto-detected). What differs is *when* the download
happens and *where* the binary lands — Composer has a post-install hook, and
Cargo, Go modules, and pip do not.

| Wrapper | Download trigger | Binary location |
|---|---|---|
| Python | `arbo-ocr-install` console script, run once after `pip install` | `arbo_ocr/bin/<platform>/` |
| Go | Lazily, on first `NewEngine` when `Config.BinPath` is empty | `os.UserCacheDir()/arbo-ocr-go/<platform>/` |
| Rust | Lazily, on first `Engine::new` when `Config.bin_path` is `None` | `%LOCALAPPDATA%\arbo-ocr-rust\<platform>` (Windows), `$XDG_CACHE_HOME/arbo-ocr-rust/<platform>` or `~/.cache/arbo-ocr-rust/<platform>` (Linux) |
| PHP | Composer post-install hook, at `composer require` / `composer install` time | `bin/<platform>/` inside the installed package |
| JavaScript | Lazily, on the first `recognize()` when `binPath` is unset | `%LOCALAPPDATA%\arbo-ocr-js\v0.3.0\<platform>` (Windows), `$XDG_CACHE_HOME/arbo-ocr-js/v0.3.0/<platform>` or `~/.cache/…` (Linux) |

Go, Rust, and JavaScript cache outside their package directories on purpose:
Go's module cache is frequently read-only, and `node_modules` is routinely
deleted and rebuilt, so neither can be written into the way Composer's
`vendor/` can.

### Pinned release

All five wrappers are pinned to release
[`v0.3.0`](https://github.com/wafik/ArboOCR/releases/tag/v0.3.0), and the
auto-download is verified working end to end on both Windows and Linux against
that tag. The model weights are pinned and cached the same way, one layer
down — see [Models](#models).

!!! warning "GPU needs v0.3.0 — every earlier archive was CPU-only in practice"
    Each wrapper exposes `useCuda` / `useTensorrt`, but the binary it downloads
    has to be able to load the provider. Release archives before v0.3.0 shipped
    no `onnxruntime_providers_shared` library, so CUDA and TensorRT could not
    load from a published package on either platform — the engine fell back to
    CPU and said nothing. The providers are `dlopen`'d at runtime, so they
    appear in no import table and `ldd` could not report them missing either.
    v0.3.0 is the first release whose archive can actually run on GPU. Read the
    `backend` field on the result to confirm which one you got.

!!! tip "Offline installs and unsupported platforms"
    If auto-download fails — air-gapped CI, a corporate proxy, an OS with no
    published asset — fetch a release manually from the
    [arboOCR releases page](https://github.com/wafik/ArboOCR/releases), unpack
    it, and pass the binary path explicitly: `bin_path` (Python, Rust),
    `Config.BinPath` (Go), or `binPath` (PHP, JavaScript). Go, Rust and
    JavaScript additionally expose their installers directly
    (`installer.EnsureInstalled`, `installer::ensure_installed`,
    `ensureInstalled()`) so you can pull the binary during a Docker or container
    build step and pass the returned path at runtime.

## Models

**No wrapper bundles OCR models — and all five auto-download them anyway.** Not
one line of wrapper code makes that happen: every wrapper spawns the same
`arboocr_demo` binary, and that binary fetches the stock weights it is missing,
so the wrappers inherit the behaviour for free. Point a models directory at
files you already have and nothing touches the network; leave them missing and
they arrive on first use. Only the recognizer has size variants; the detector is
always one file, and the angle classifier is only needed if you turn angle
classification on.

| File | Needed for | Varies by model type? |
|---|---|---|
| `PP-OCRv6_det.onnx` | Detection | No — always this one file |
| `PP-OCRv6_rec_tiny.onnx` + `PP-OCRv6_rec_tiny_dict.txt` | Model type `tiny` | Yes |
| `PP-OCRv6_rec_small.onnx` + `PP-OCRv6_rec_small_dict.txt` | Model type `small` (default) | Yes |
| `PP-OCRv6_rec_medium.onnx` + `PP-OCRv6_rec_medium_dict.txt` | Model type `medium` | Yes |
| `PP-OCRv6_cls.onnx` | Angle classification — only when it is enabled | No |

You only need the recognizer size(s) you will actually use. For the default
`small` alone, the models directory needs just `PP-OCRv6_det.onnx`,
`PP-OCRv6_rec_small.onnx`, and `PP-OCRv6_rec_small_dict.txt`. Switching sizes
later is a one-word config change; the directory can hold all three sizes side
by side so you can switch freely at runtime.

### Four ways to get the files

1. **Do nothing.** This is the default now, which is what turns the other three
   into deliberate choices rather than prerequisites. The first time a wrapper
   spawns `arboocr_demo` with stock weights missing, the engine downloads them,
   checks each file against a SHA-256 baked into the binary, and writes it
   atomically. A mirror serving different bytes
   is rejected rather than loaded, and a half-finished download never becomes a
   file the next run mistakes for complete.
2. **Copy from a `rapidocr` install.** If you already have the Python
   `rapidocr` package, copy its `models/` directory over and rename the files
   to match the layout above.
3. **Use your own PP-OCRv6 ONNX export.** Place and rename the files as above.
4. **Use a local arboOCR checkout.** Its `models/` directory already contains
   the detector, the classifier, and all three recognizer sizes — the fastest
   path for local development.

Options 2–4 all reduce to the same rule: **a populated models directory wins.**
If the files the engine wants are already sitting in `models_dir` / `ModelsDir`
/ `modelsDir`, there is no network access at all.

Where the downloaded files land — tag-scoped, so a future `models-v2` can never
reuse a `models-v1` file:

| Platform | Model cache directory |
|---|---|
| Windows | `%LOCALAPPDATA%\arboOCR\models\models-v1` |
| macOS | `~/Library/Caches/arboOCR/models/models-v1` |
| Linux | `$XDG_CACHE_HOME/arboOCR/models/models-v1`, falling back to `~/.cache/arboOCR/models/models-v1` |

That is the same shape as [Getting the binary](#getting-the-binary) above: a
pinned tag, a platform cache directory, a lazy fetch on first use. Same pattern,
second artifact — the wrappers cache the CLI under their own name, and the CLI
caches the weights under arboOCR's.

!!! tip "Turning the download off from any of the five languages"
    Only the JavaScript wrapper exposes `arboocr_demo`'s `--no-download` and
    `--models-url` flags directly (as `noDownload` and `modelsUrl`). Everywhere
    else, a spawned process inherits your environment, so the env vars reach it
    from every language. `ARBOOCR_OFFLINE=1` makes a
    missing model an immediate error instead of a network call — the setting you
    want in an air-gapped runtime, where a stalled socket is indistinguishable
    from a hang. `ARBOOCR_MODELS_URL` points at an internal mirror, and
    `ARBOOCR_CACHE_DIR` moves the cache somewhere writable, which matters when
    your service runs as a user with no home directory. To prefetch during a
    Docker build instead of at first request, run the binary once with
    `--download-models`; it downloads and exits.

arboOCR does host a default download URL now:
[`models-v1`](https://github.com/ARBO-TEAM/arbo-ocr-models/releases/tag/models-v1)
in [ARBO-TEAM/arbo-ocr-models](https://github.com/ARBO-TEAM/arbo-ocr-models), an
immutable release tag rather than a branch, with every stock file checksummed.
See [Models](../models/index.md) for the full treatment, including
[size trade-offs](../models/sizes.md) and
[language coverage](../models/languages.md).

## Overhead

All wrappers call the identical `arboocr_demo` binary, so **recognition
accuracy is the same across all four** — 83.7% / 85.3% / 85.5% for
tiny / small / medium on the reference SROIE sample. The numbers below measure
wrapper overhead only: subprocess spawn cost minus the engine's own reported
inference time, over a 40-image SROIE sample.

| Model size | Rust | Go | JavaScript | PHP | Python |
|---|---:|---:|---:|---:|---:|
| `tiny` | 135 ms | 174 ms | 186 ms | 196 ms | 218 ms |
| `small` | 162 ms | 177 ms | 216 ms | 217 ms | 246 ms |
| `medium` | 233 ms | 235 ms | 282 ms | 291 ms | 317 ms |

The raw binary — no wrapper at all, just `arboocr_demo` spawned directly —
costs 127 / 184 / 234 ms on the same run. That is the floor every row is
measured against, and it is most of every row.

Go and Rust track each other closely: both are compiled binaries paying only
process-spawn cost. JavaScript sits between them and the interpreted wrappers,
since Node's startup is cheaper than PHP's or Python's but not free. PHP and
Python land in the same ballpark as one another.

Overhead rises with model size for every language, which is the tell that this
is not purely spawn cost — a larger recognizer takes longer to load into memory,
and that load happens inside the process on every call without appearing in the
engine's own reported time.

!!! note "Pick on ecosystem, not on this table"
    A 60 ms spread is noise next to the 130–320 ms floor every wrapper shares.
    If that floor is the problem, no wrapper fixes it — go in-process instead.

## Next steps

- [Python](python.md) — `pip install` + `arbo-ocr-install`
- [Go](go.md) — `go get`, lazy binary download
- [Rust](rust.md) — Cargo git dependency
- [PHP](php.md) — Composer with a post-install hook
- [JavaScript](js.md) — npm or Bun, zero runtime dependencies
- [Models](../models/index.md) — what to download and where to put it
- [Quickstart](../quickstart.md) — the CLI, C++, and in-process Python bindings
