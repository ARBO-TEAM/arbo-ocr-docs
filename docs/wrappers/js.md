---
title: JavaScript
---

# JavaScript (Node.js & Bun)

JavaScript wrapper for arboOCR. It runs the prebuilt `arboocr_demo` binary via
`child_process` — **no C++ build required**, no native module, no `node-gyp`.

- **Package:** `arbo-ocr-js`
- **Repo:** [ARBO-TEAM/arbo-ocr-js](https://github.com/ARBO-TEAM/arbo-ocr-js)
- **Runtime:** Node.js ≥ 18, or Bun ≥ 1.0
- **Config field style:** `camelCase`
- **Runtime dependencies:** none

## Install

```bash
npm install github:ARBO-TEAM/arbo-ocr-js
# or
bun add github:ARBO-TEAM/arbo-ocr-js
```

!!! note "Not on the npm registry yet"
    Install from GitHub for now. The package's `prepare` script builds `dist/`
    at install time, so a git install behaves exactly like a registry one — you
    still need no compiler and no `node-gyp`.

The first `recognize()` call downloads the matching arboOCR release binary
(Windows or Linux, auto-detected). The pinned release is
[`v0.3.0`](https://github.com/wafik/ArboOCR/releases/tag/v0.3.0), and the
auto-download is verified in CI on both platforms and both runtimes.

!!! tip "If the download fails"
    On an offline build or an unsupported OS, download a release manually from
    the [arboOCR releases page](https://github.com/wafik/ArboOCR/releases) and
    pass `binPath` explicitly — see [Manual binary path](#manual-binary-path).

## Usage

```ts
import { Engine } from "arbo-ocr-js";

const engine = new Engine({
  modelsDir: "/path/to/models",
  modelType: "small",   // tiny/small/medium — default small
  // useAngleCls: true,
  // useCuda: true,
});

const page = await engine.recognize("/path/to/image.jpg");

console.log(page.backend); // cpu / cuda / tensorrt
for (const line of page.lines) {
  console.log(line.text, line.score.toFixed(3));
}
```

Construction is synchronous and touches nothing — no network, no filesystem. The
binary is resolved on the first `recognize()` or `ensureModels()` call, and the
promise is memoized, so several concurrent first calls share one download rather
than racing.

## Models

`arbo-ocr-js` does not bundle OCR models — it does not have to. The
`arboocr_demo` binary it spawns downloads the stock weights it is missing on
first use, checks each file against a SHA-256 baked into the binary, and caches
it per platform. Point `modelsDir` at a folder that already holds the PP-OCRv6
ONNX files and nothing touches the network. For the default `small` that means
three files:

```text
models/
├── PP-OCRv6_det.onnx
├── PP-OCRv6_rec_small.onnx
└── PP-OCRv6_rec_small_dict.txt
```

`PP-OCRv6_cls.onnx` is only needed when `useAngleCls` is on.

To prefetch during a container build so the first request never waits:

```ts
await new Engine({ modelType: "small" }).ensureModels();
```

Full file matrix, the cache locations, and how to turn the download off:
[Models](index.md#models) on the wrappers overview, or
[Models](../models/index.md) for the complete treatment.

## Configuration

Every field is optional.

| Field | Type | Default | Description |
|---|---|---|---|
| `modelsDir` | `string` | — | Directory holding the PP-OCRv6 ONNX files |
| `binPath` | `string` | auto-download | Explicit path to `arboocr_demo` |
| `modelType` | `string` | `"small"` | Recognizer size: `tiny`, `small`, `medium` |
| `ocrVersion` | `string` | `"PP-OCRv6"` | Model family |
| `detModelPath` `clsModelPath` `recModelPath` `dictPath` | `string` | — | Per-file overrides; never substituted by a download |
| `logLevel` | `string` | silent | `debug`, `info`, `warn`, `error` — engine events on stderr |
| `minConfidence` | `number` | `0.5` | Drop lines below this score; `0` disables filtering |
| `recBatchNum` | `number` | `6` | Crops per recognition inference call |
| `detLimitSideLen` | `number` | `960` | Longest image side for detection resize |
| `useAngleCls` | `boolean` | `false` | Angle classification (requires `PP-OCRv6_cls.onnx`) |
| `useCuda` | `boolean` | `false` | Request the CUDA execution provider |
| `useTensorrt` | `boolean` | `false` | Request the TensorRT execution provider |
| `useFp16` | `boolean` | `true` | TensorRT FP16; only used with `useTensorrt` |
| `useClahe` | `boolean` | `false` | CLAHE contrast enhancement before detection |
| `wordBoxes` | `boolean` | `false` | Adds `line.words` — a polygon per word |
| `noDownload` | `boolean` | `false` | Fail instead of fetching a missing model |
| `modelsUrl` | `string` | — | Fetch missing models from an internal mirror |

### Unset means unset

An omitted field emits **no CLI flag at all**, leaving the binary's own default
in place. That is a deliberate difference from the Go wrapper, which cannot
represent "unset" for a `bool` and so always emits `--angle=true|false`.

It buys real compatibility: point `binPath` at a pre-v0.3.0 binary and the flags
that release never had — `--no-download`, `--models-url`, `--word-boxes` — are
simply never sent. `cxxopts` exits 1 on an unknown option, so a flag that is
never emitted can never break an older binary.

### Manual binary path

Setting `binPath` suppresses the auto-download entirely.

```ts
const engine = new Engine({ binPath: "/opt/arboocr/arboocr_demo" });
```

Or drive the installer yourself — useful in a Docker build step, so runtime
never touches the network:

```ts
import { ensureInstalled, detectPlatform } from "arbo-ocr-js";

if (!detectPlatform()) throw new Error("no release asset for this platform");
const binPath = await ensureInstalled();
```

!!! note "The models download is the binary's job, not this package's"
    `ensureInstalled` gets you the binary and *not* the models, so the first
    request would still reach for the network. Prefetch both in the same step by
    also calling `engine.ensureModels()`, or set `ARBOOCR_OFFLINE=1` at runtime
    so a missing model fails fast instead of opening a socket.

## Result shape

```ts
interface PageResult {
  backend: string;      // "cpu" | "cuda" | "tensorrt" — what actually ran
  image: string;
  elapsedMs: number;    // engine-reported inference time
  lines: LineResult[];  // in detection order
}

interface LineResult {
  text: string;
  score: number;        // recognition confidence, 0.0–1.0
  detScore: number;     // detection confidence
  polygon: Point[];     // 4 points, clockwise from top-left-ish
  words?: WordResult[]; // only when wordBoxes is set
}
```

## Error handling

`recognize()` rejects with an `OcrError` — carrying `exitCode` and `stderr` —
only when the process fails to start, exits non-zero, or emits output that is
not JSON.

```ts
import { Engine, OcrError } from "arbo-ocr-js";

try {
  const page = await engine.recognize("scan.png");
} catch (err) {
  if (err instanceof OcrError) console.error(err.exitCode, err.stderr);
  else throw err;
}
```

!!! note "Empty results are not errors"
    An empty `page.lines` array means no text was found. That is a successful
    call with zero detections, not a failure. Check `page.lines.length`, not the
    rejection, to distinguish "no text" from "broken pipeline".

## Bun

The package is plain ESM built by `tsc`, with no native module and no
Bun-specific code, so the same `dist/` runs on both runtimes. CI exercises Node
and Bun on Linux and Windows against a real download and a real image.

Bun is not faster here in any way that matters: the cost is dominated by the
subprocess, not by the JavaScript runtime around it.

!!! question "Why not `bun:ffi` instead of a subprocess?"
    It is a fair question, and the answer is not the obvious one — see
    [In-process bindings](#in-process-bindings) below.

## How it works

The package never builds or vendors arboOCR's C++ source. It downloads the exact
same release asset every other wrapper uses — `arboocr-windows-x64.zip` or
`arboocr-linux-x64.tar.gz` — and runs the same CLI tool via `child_process`
instead of Go's `os/exec` or PHP's `proc_open`.

### Zero runtime dependencies

An npm package that downloads and unpacks an archive would normally pull in a
HTTP client and two archive libraries. This one pulls in nothing:

| Job | What it uses instead |
|---|---|
| Download | The global `fetch`, built into Node 18+ and Bun |
| Extract | `tar -xf`, shelled out |
| Types | `tsc` only — no bundler, no dual CJS build |

The extraction choice is the interesting one. GNU `tar` handles the Linux
`.tar.gz`, and Windows has shipped **bsdtar** (libarchive) in `System32` since
Windows 10 1803 — which reads the `.zip` too. One tool covers both release
formats.

!!! warning "Why the Windows path is absolute"
    The wrapper invokes `%SystemRoot%\System32\tar.exe` by full path, not `tar`
    by `PATH`. Git for Windows puts GNU tar on `PATH`, and **GNU tar cannot read
    a zip file**. Resolving by `PATH` would work or fail depending on which
    shell the user happened to install — the worst kind of bug to receive a
    report about.

### Binary cache

| Platform | Location |
|---|---|
| Windows | `%LOCALAPPDATA%\arbo-ocr-js\v0.3.0\windows-x64\` |
| Linux | `$XDG_CACHE_HOME/arbo-ocr-js/v0.3.0/linux-x64/` (or `~/.cache/…`) |

The version is a path segment on purpose. The extracted binary has the same name
in every release, so a version-less path would be the same path for every
version — and the "already installed" check would report a v0.1.0 binary as
current forever, making a version bump a silent no-op for everyone who had ever
run an older pin.

### Interrupted installs

The install stages into a temp directory and moves files into place with the
**executable last**. Since "already installed" is a `stat` on the executable,
publishing it before its DLLs would let a half-finished download look complete
and then fail to load. Moving it last makes its presence a genuine completion
marker.

### Deadlock avoidance

`execFile` drains stdout and stderr concurrently. `arboocr_demo` can write
~200 KB of ONNX Runtime warnings to stderr before any stdout appears — enough to
deadlock a reader that consumes one stream to completion before touching the
other. `maxBuffer` is raised to 64 MB, well past Node's 1 MB default, which
`--word-boxes` output on a dense page can otherwise exceed.

## In-process bindings

`bun:ffi` looks like the obvious upgrade, and it is worth stating plainly why
this package does not use it.

**It is not needed for GPU.** CUDA and TensorRT already work through the
subprocess — `arboocr_demo` loads the same ONNX Runtime providers either way,
and v0.3.0 is the release that ships `onnxruntime_providers_shared`. Set
`useCuda: true` and read `page.backend`. FFI changes nothing here.

**The pybind11 bindings cannot be reused.** They emit a *Python extension
module*, which exports Python C-API symbols. `bun:ffi` can only call a plain C
ABI — `extern "C"`, primitives and pointers. It cannot call a C++ class, receive
a `std::string`, or load a `.pyd`. A shim would be new code, not reuse.

**What FFI would actually buy** is not raw call speed — inference runs 750 ms to
4 s per image, against 130–320 ms of spawn overhead. It buys back the spawn and,
more importantly, the **model reload** on every call. It would also allow passing
a `Uint8Array` straight in: `Engine::recognize(const cv::Mat&)` and
`recognizeEncoded(const uint8_t*, size_t)` both already exist, and the latter is
already a C-ABI-shaped signature.

**What it would cost:** ~50 MB of transitive DLLs that `dlopen` must locate on
every platform, a second CI packaging matrix, a permanently versioned ABI, and
crashes that become segfaults inside your process instead of a non-zero exit you
can catch. It would also be Bun-only — Node would need Koffi or a Node-API
addon.

Model load is the real fixed cost, and the CLI's `--images-from` batch mode
already amortizes it across many images with one Engine construction. For "user
uploads a receipt, we OCR it", the subprocess is the correct call. FFI earns its
keep only when you need per-call latency under ~50 ms on already-decoded frames —
a video or camera loop you genuinely cannot batch.

## Performance

Wrapper overhead only — subprocess spawn minus the engine's own reported
inference time, over a 40-image SROIE sample. Accuracy is identical across all
wrappers because they call the same binary.

| Model size | JavaScript | Rust | Go | PHP | Python |
|---|---:|---:|---:|---:|---:|
| `tiny` | 186 ms | 135 ms | 174 ms | 196 ms | 218 ms |
| `small` | 216 ms | 162 ms | 177 ms | 217 ms | 246 ms |
| `medium` | 282 ms | 233 ms | 235 ms | 291 ms | 317 ms |

JavaScript sits between the compiled wrappers and the interpreted ones: Node's
startup is cheaper than PHP's or Python's, and not free the way Go's and Rust's
are. Against the raw binary spawned with no wrapper at all — 127 / 184 / 234 ms
on the same run — the JavaScript layer itself accounts for roughly 30–60 ms.

Bun is not measurably different here. The cost is the subprocess, not the
runtime around it.

## License

Apache-2.0.

## See also

- [Language wrappers](index.md) — shared architecture and the trade-off
- [Go](go.md) — the closest analogue, same lazy-download design
- [Models](../models/index.md) — obtaining PP-OCRv6 ONNX files
