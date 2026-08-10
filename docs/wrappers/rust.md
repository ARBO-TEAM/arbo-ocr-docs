---
title: Rust
---

# Rust

Rust wrapper for arboOCR. It runs the prebuilt `arboocr_demo` binary via
`std::process::Command` — **no C++ build required**, no `build.rs` linking, no
FFI.

- **Crate:** `arbo-ocr` (git dependency)
- **Import path:** `arbo_ocr`
- **Repo:** [ARBO-TEAM/arbo-ocr-rust](https://github.com/ARBO-TEAM/arbo-ocr-rust)
- **Config field style:** `snake_case`

## Install

```toml
[dependencies]
arbo-ocr = { git = "https://github.com/ARBO-TEAM/arbo-ocr-rust" }
```

`Engine::new` downloads the matching arboOCR release binary (Windows or Linux,
auto-detected) the first time it is used **if `Config.bin_path` is `None`**.
Verified working end to end against
[`v0.3.0`](https://github.com/wafik/ArboOCR/releases/tag/v0.3.0) on both
platforms.

!!! tip "If the download fails"
    On an offline build or an unsupported OS, download a release manually from
    the [arboOCR releases page](https://github.com/wafik/ArboOCR/releases) and
    pass `Config.bin_path` explicitly — see
    [Manual binary path](#manual-binary-path).

## Models

`arbo-ocr` does not bundle OCR models — it does not have to. The `arboocr_demo`
binary it spawns downloads the stock weights it is missing on first use, checks
each file against a SHA-256 baked into the binary, and caches it per platform.
Point `Config.models_dir` at a folder that already holds the PP-OCRv6 ONNX files
and nothing touches the network. For the default `model_type: "small"` that
means three files:

```text
models/
├── PP-OCRv6_det.onnx
├── PP-OCRv6_rec_small.onnx
└── PP-OCRv6_rec_small_dict.txt
```

`PP-OCRv6_cls.onnx` is only needed when `use_angle_cls` is on. Swap the `_small`
files for `_tiny` or `_medium` to change size — or keep all three in the same
directory and switch with `model_type`.

Full file matrix, the cache locations, and how to turn the download off:
[Models](index.md#models) on the wrappers overview, or
[Models](../models/index.md) for the complete treatment.

## Usage

```rust
use arbo_ocr::{Config, Engine};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let engine = Engine::new(Config {
        models_dir: Some("/path/to/models".to_string()),
        // bin_path: Some("/custom/path/to/arboocr_demo".into()), // optional override
        // model_type: Some("small".to_string()), // tiny/small/medium — default small
        // use_angle_cls: true,
        // use_cuda: true,
        ..Default::default()
    })?;

    let result = engine.recognize("/path/to/image.jpg")?;

    println!("{}", result.backend); // cpu / cuda / tensorrt
    for line in &result.lines {
        println!("{} ({:.3})", line.text, line.score);
    }
    Ok(())
}
```

!!! note "`..Default::default()` is the idiomatic form"
    `Config` implements `Default`, so you only spell out the fields you care
    about and let the struct-update syntax fill in the rest. Omitting
    `..Default::default()` forces you to name every field.

## Configuration

`arbo_ocr::Config` fields:

| Field | Type | Default | Description |
|---|---|---|---|
| `models_dir` | `Option<String>` | `None` | Directory holding the PP-OCRv6 ONNX files |
| `bin_path` | `Option<_>` (path) | `None` | Explicit path to `arboocr_demo`. `None` triggers the lazy auto-download |
| `model_type` | `Option<String>` | `None` → `small` | Recognizer size: `"tiny"`, `"small"`, or `"medium"` |
| `use_angle_cls` | `bool` | `false` | Enable angle classification (requires `PP-OCRv6_cls.onnx`) |
| `use_cuda` | `bool` | `false` | Request the CUDA execution provider |

### Manual binary path

`Config.bin_path` is an `Option`. **`None` is what triggers the auto-download** —
set it to `Some(path)` and `Engine::new` skips the network entirely and uses the
binary you point at:

```rust
let engine = Engine::new(Config {
    models_dir: Some("/path/to/models".to_string()),
    bin_path: Some("/opt/arboocr/arboocr_demo".into()),
    ..Default::default()
})?;
```

## Error handling

`Engine::recognize` returns `Err(OcrError)` only when the process itself fails
to start, exits non-zero, or produces unparseable output.

!!! note "Empty results are not errors"
    An empty `result.lines` vec means no text was found in the image. That is
    `Ok` with zero detections, not an `Err`. Check `result.lines.is_empty()`,
    not the `Result`, to distinguish "no text" from "broken pipeline".

## Result shape

`Engine::recognize` returns a result struct with these fields:

| Field | Type | Description |
|---|---|---|
| `backend` | `String` | Execution provider actually used: `cpu`, `cuda`, or `tensorrt` |
| `lines` | `Vec<_>` | Recognized lines, in detection order |
| `elapsed_ms` | float | Engine-reported inference time in milliseconds |

Each element of `lines` carries:

| Field | Type | Description |
|---|---|---|
| `text` | `String` | Recognized text for the line |
| `score` | float | Recognition confidence, `0.0`–`1.0` |

## Quick example: tiny model

For a fast local smoke test, use `model_type: "tiny"` — the smallest and fastest
PP-OCRv6 recognizer. A local arboOCR checkout's `models/` folder already
contains the tiny det/rec/cls ONNX files, so no extra download is needed:

```rust
use arbo_ocr::{Config, Engine};

let engine = Engine::new(Config {
    models_dir: Some("/path/to/arboOCR/models".to_string()), // e.g. a local arboOCR checkout's models/ dir
    model_type: Some("tiny".to_string()),
    ..Default::default()
})?;

let result = engine.recognize("/path/to/receipt.jpg")?;

println!(
    "backend={} lines={} elapsedMs={:.1}",
    result.backend,
    result.lines.len(),
    result.elapsed_ms
);
for line in &result.lines {
    println!("  {:<40} score={:.3}", line.text, line.score);
}
```

`tiny` trades accuracy for speed — good for quick local testing. Switch to
`small` (the default) or `medium` for production-quality recognition.

## How it works

The crate never builds or vendors arboOCR's C++ source. It downloads the exact
same prebuilt release asset the PHP and Go packages use —
`arboocr-windows-x64.zip` or `arboocr-linux-x64.tar.gz` from the
[wafik/ArboOCR releases](https://github.com/wafik/ArboOCR/releases). The
compiled binary and its DLLs are language-agnostic; this crate just runs the
same CLI tool via `std::process::Command`.

### Why the download is lazy

Cargo has no build-time install hook like Composer's `post-install-cmd`, so
`Engine::new` downloads lazily instead — the same design as
[arbo-ocr-go](go.md). It calls `installer::ensure_installed(None)` when
`Config.bin_path` is `None`, caching the result under the OS user cache
directory:

| Platform | Cache location |
|---|---|
| Windows | `%LOCALAPPDATA%\arbo-ocr-rust\<platform>` |
| Linux | `$XDG_CACHE_HOME/arbo-ocr-rust/<platform>`, falling back to `~/.cache/arbo-ocr-rust/<platform>` |

### Controlling when the download happens

```rust
// e.g. in a container build step, so runtime never touches the network
let bin = arbo_ocr::installer::ensure_installed(Some(dir))?;
```

Call `installer::ensure_installed(Some(dir))` yourself to control exactly when
the download happens, then pass the returned path as `Config.bin_path`.

!!! note "The models download is the binary's job, not this crate's"
    `arbo-ocr` downloads exactly one thing: the `arboocr_demo` binary. The
    weights are fetched separately, by that binary, into arboOCR's own platform
    cache — so `installer::ensure_installed` in a container build step gets you
    the binary and *not* the models, and the first request would still reach for
    the network. Prefetch both in the same step by running the returned binary
    once with `--download-models`, or set `ARBOOCR_OFFLINE=1` at runtime so a
    missing model fails fast instead of opening a socket. See
    [Models](../models/index.md).

### Deadlock avoidance

`recognize` captures the subprocess's output with `Command::output()`, which
reads stdout and stderr concurrently on separate threads internally.
`arboocr_demo` can write ~200 KB of ONNXRuntime warnings to stderr before any
stdout appears — enough to deadlock a naive sequential-pipe-read implementation,
a class of bug the PHP and Go wrappers had to explicitly design around.

### Bool flags use `--flag=value`

Boolean options (`use_angle_cls`, `use_cuda`, `use_tensorrt`, `use_fp16`,
`use_clahe`) are always sent as a single `--flag=value` token, never a bare
`--flag` followed by a separate `true`/`false` token.

!!! warning "Why this matters"
    `arboocr_demo`'s argument parser (cxxopts) only binds a bool flag's value
    via `=`. The two-token form leaves the flag implicitly `true` regardless of
    the intended value. This exact bug was found and fixed in both
    [arbo-ocr-php](php.md) and [arbo-ocr-go](go.md) after it crashed
    `arboocr_demo` in production; this crate avoids it from the start.

## Performance

Wrapper overhead only — subprocess spawn minus the engine's own reported
inference time. Accuracy is identical across all wrappers because they call the
same binary.

| Model size | Rust | Go | JavaScript | PHP | Python |
|---|---:|---:|---:|---:|---:|
| `tiny` | 135 ms | 174 ms | 186 ms | 196 ms | 218 ms |
| `small` | 162 ms | 177 ms | 216 ms | 217 ms | 246 ms |
| `medium` | 233 ms | 235 ms | 282 ms | 291 ms | 317 ms |

Rust and Go are effectively tied — both are compiled binaries paying only
process-spawn cost, with no interpreter startup.

## License

Apache-2.0.

## See also

- [Language wrappers](index.md) — shared architecture and the trade-off
- [Go](go.md) — the closest analogue, same lazy-download design
- [Models](../models/index.md) — obtaining PP-OCRv6 ONNX files
