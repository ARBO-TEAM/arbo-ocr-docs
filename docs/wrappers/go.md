---
title: Go
---

# Go

Go wrapper for arboOCR. It runs the prebuilt `arboocr_demo` binary via
`os/exec` — **no C++ build required**, no cgo.

- **Module:** `github.com/ARBO-TEAM/arbo-ocr-go`
- **Import alias:** `arboocr`
- **Repo:** [ARBO-TEAM/arbo-ocr-go](https://github.com/ARBO-TEAM/arbo-ocr-go)
- **Config field style:** `PascalCase`

## Install

```bash
go get github.com/ARBO-TEAM/arbo-ocr-go
```

`NewEngine` downloads the matching arboOCR release binary (Windows or Linux,
auto-detected) the first time it is used. The pinned release is
[`v0.3.0`](https://github.com/wafik/ArboOCR/releases/tag/v0.3.0), and the
auto-download is live and verified working end to end — no manual binary step
needed.

!!! tip "If the download fails"
    On an offline build or an unsupported OS, download a release manually from
    the [arboOCR releases page](https://github.com/wafik/ArboOCR/releases) and
    pass `Config.BinPath` explicitly — see
    [Manual binary path](#manual-binary-path).

## Models

`arbo-ocr-go` does not bundle OCR models — it does not have to. The
`arboocr_demo` binary it spawns downloads the stock weights it is missing on
first use, checks each file against a SHA-256 baked into the binary, and caches
it per platform. Point `Config.ModelsDir` at a folder that already holds the
PP-OCRv6 ONNX files and nothing touches the network. For the default
`ModelType: "small"` that means three files:

```text
models/
├── PP-OCRv6_det.onnx
├── PP-OCRv6_rec_small.onnx
└── PP-OCRv6_rec_small_dict.txt
```

`PP-OCRv6_cls.onnx` is only needed when `UseAngleCls` is on. Swap the `_small`
files for `_tiny` or `_medium` to change size — or keep all three in the same
directory and switch with `ModelType`.

Full file matrix, the cache locations, and how to turn the download off:
[Models](index.md#models) on the wrappers overview, or
[Models](../models/index.md) for the complete treatment.

## Usage

```go
package main

import (
	"fmt"
	"log"

	arboocr "github.com/ARBO-TEAM/arbo-ocr-go"
)

func main() {
	engine, err := arboocr.NewEngine(arboocr.Config{
		ModelsDir: "/path/to/models",
		// BinPath:     "/custom/path/to/arboocr_demo", // optional override
		// ModelType:   "small", // tiny/small/medium — default small
		// UseAngleCls: true,
		// UseCuda:     true,
	})
	if err != nil {
		log.Fatal(err)
	}

	result, err := engine.Recognize("/path/to/image.jpg")
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(result.Backend) // cpu / cuda / tensorrt
	for _, line := range result.Lines {
		fmt.Printf("%s (%.3f)\n", line.Text, line.Score)
	}
}
```

## Configuration

`arboocr.Config` fields:

| Field | Type | Default | Description |
|---|---|---|---|
| `ModelsDir` | `string` | — | Directory holding the PP-OCRv6 ONNX files |
| `BinPath` | `string` | `""` (auto-download) | Explicit path to `arboocr_demo`. Leave empty to trigger the lazy download |
| `ModelType` | `string` | `"small"` | Recognizer size: `"tiny"`, `"small"`, or `"medium"` |
| `UseAngleCls` | `bool` | `false` | Enable angle classification (requires `PP-OCRv6_cls.onnx`) |
| `UseCuda` | `bool` | `false` | Request the CUDA execution provider |

### Manual binary path

`Config.BinPath` is the manual override. Setting it suppresses the auto-download
entirely — `NewEngine` only triggers a download when `BinPath` is left empty.

```go
engine, err := arboocr.NewEngine(arboocr.Config{
	ModelsDir: "/path/to/models",
	BinPath:   "/opt/arboocr/arboocr_demo",
})
```

## Error handling

`Engine.Recognize` returns a non-nil error — of type `*arboocr.OcrError` — only
when the process itself fails to start, exits non-zero, or produces unparseable
output.

!!! note "Empty results are not errors"
    An empty `result.Lines` slice means no text was found in the image. That is
    a successful call with zero detections, not a failure. Check `len(result.Lines)`,
    not `err`, to distinguish "no text" from "broken pipeline".

## Result shape

`Engine.Recognize` returns a result struct with these fields:

| Field | Type | Description |
|---|---|---|
| `Backend` | `string` | Execution provider actually used: `cpu`, `cuda`, or `tensorrt` |
| `Lines` | slice | Recognized lines, in detection order |
| `ElapsedMs` | float | Engine-reported inference time in milliseconds |

Each element of `Lines` carries:

| Field | Type | Description |
|---|---|---|
| `Text` | `string` | Recognized text for the line |
| `Score` | float | Recognition confidence, `0.0`–`1.0` |

## Quick example: tiny model

For a fast local smoke test, use `ModelType: "tiny"` — the smallest and fastest
PP-OCRv6 recognizer. A local arboOCR checkout's `models/` folder already
contains the tiny det/rec/cls ONNX files, so no extra download is needed:

```go
engine, err := arboocr.NewEngine(arboocr.Config{
	ModelsDir: "/path/to/arboOCR/models", // e.g. a local arboOCR checkout's models/ dir
	ModelType: "tiny",
})
if err != nil {
	log.Fatal(err)
}

result, err := engine.Recognize("/path/to/receipt.jpg")
if err != nil {
	log.Fatal(err)
}

fmt.Printf("backend=%s lines=%d elapsedMs=%.1f\n", result.Backend, len(result.Lines), result.ElapsedMs)
for _, line := range result.Lines {
	fmt.Printf("  %-40s score=%.3f\n", line.Text, line.Score)
}
```

`tiny` trades accuracy for speed — good for quick local testing. Switch to
`small` (the default) or `medium` for production-quality recognition.

## How it works

The package never builds or vendors arboOCR's C++ source. It downloads the exact
same prebuilt release asset the PHP package uses — `arboocr-windows-x64.zip` or
`arboocr-linux-x64.tar.gz` from the
[wafik/ArboOCR releases](https://github.com/wafik/ArboOCR/releases). The
compiled binary and its DLLs are language-agnostic; this package just runs the
same CLI tool via `os/exec` instead of PHP's `proc_open`.

### Why the download is lazy

Composer has a post-install hook that pulls the binary into the project's own
`vendor/` directory at `composer install` time. Go modules have no equivalent
build-time hook, so `arbo-ocr-go` downloads lazily instead: `NewEngine` triggers
a download only when `Config.BinPath` is empty.

The download is handled by a separate `installer` package — kept out of the root
package so "how to get the binary" stays independent of "how to run it". It
caches under the OS user cache directory,
`os.UserCacheDir()/arbo-ocr-go/<platform>/`, rather than anywhere inside the
module itself: Go's module cache is often read-only, so it cannot be written
into the way Composer's `vendor/` can.

### Controlling when the download happens

```go
// e.g. in a Docker build step, so runtime never touches the network
binPath, err := installer.EnsureInstalled(binDir)
```

`installer.DetectPlatform` reports which release asset matches the current
OS/arch. Call `installer.EnsureInstalled(binDir)` yourself to control exactly
when the download happens, then pass the returned path as `Config.BinPath`.

!!! note "The models download is the binary's job, not this package's"
    `arbo-ocr-go` downloads exactly one thing: the `arboocr_demo` binary. The
    weights are fetched separately, by that binary, into arboOCR's own platform
    cache — so `installer.EnsureInstalled` in a Docker build step gets you the
    binary and *not* the models, and the first request would still reach for the
    network. Prefetch both in the same step by running the returned binary once
    with `--download-models`, or set `ARBOOCR_OFFLINE=1` at runtime so a missing
    model fails fast instead of opening a socket. See
    [Models](../models/index.md).

### Deadlock avoidance

`Recognize` captures the subprocess's output with a buffered `exec.Cmd.Run()` —
`cmd.Stdout` and `cmd.Stderr` set to plain `io.Writer` values, which Go's stdlib
drains concurrently — rather than sequential pipe reads. `arboocr_demo` can
write ~200 KB of ONNXRuntime warnings to stderr before any stdout appears, which
is enough to deadlock a naive pipe-based reader. The PHP wrapper had to
explicitly work around this class of bug.

## Performance

Wrapper overhead only — subprocess spawn minus the engine's own reported
inference time. Accuracy is identical across all wrappers because they call the
same binary.

| Model size | Go | Rust | PHP | Python |
|---|---:|---:|---:|---:|
| `tiny` | 138 ms | 131 ms | 192 ms | 203 ms |
| `small` | 184 ms | 174 ms | 234 ms | 249 ms |
| `medium` | 253 ms | 245 ms | 302 ms | 321 ms |

Go and Rust are effectively tied — both are compiled binaries paying only
process-spawn cost, with no interpreter startup.

## License

Apache-2.0.

## See also

- [Language wrappers](index.md) — shared architecture and the trade-off
- [Rust](rust.md) — the closest analogue, same lazy-download design
- [Models](../models/index.md) — obtaining PP-OCRv6 ONNX files
