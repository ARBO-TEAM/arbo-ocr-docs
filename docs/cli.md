---
title: Command line
description: >-
  Full flag reference for arboocr_demo — the binary every arboOCR language
  wrapper spawns. Defaults, batch input, exit codes, the --json contract, and
  tuning recipes.
---

# Command line

`arboocr_demo` is the binary that ships in every arboOCR release. It looks like
a demo, and for a C++ user it is one. For everyone else it is **the public
API**: all four wrappers — [Python](wrappers/python.md), [Go](wrappers/go.md),
[Rust](wrappers/rust.md), [PHP](wrappers/php.md) — spawn this exact executable
as a subprocess and parse its stdout. `subprocess`, `os/exec`,
`std::process::Command`, `proc_open`: different words, one binary.

That is why this page matters more than a demo page should. Until recently the
CLI exposed no accuracy knobs at all — no detection thresholds, no
`--min-confidence`, no `--clahe`. Every wrapper was therefore locked out of the
library's own tuning story: you could read
[Accuracy defaults](models/accuracy-defaults.md), agree with all of it, and
have no way to act on it from Python. The flags below close that gap. A knob
that exists here exists in all four languages.

```text
arboocr_demo --image <path>        [options]
arboocr_demo --images-from <path>  [options]
arboocr_demo --download-models     [options]
```

Exactly one input flag is required. Without either, the binary prints usage and
exits `1`; passing **both** is an error. `--image` recognizes one file;
[`--images-from`](#batch-input) reads a list of paths and recognizes all of them
in a single process — see the next section for why that is not just a
convenience. The third form recognizes nothing at all: it
[prewarms the model cache](#prewarming-the-cache-download-models) and exits, so
it needs no input.

!!! note "Boolean flags"

    Every `bool` option is a presence switch: writing `--clahe` sets it true.
    The one that needs care is `--fp16`, which **defaults to true** — to turn it
    off you must write `--fp16=false`, not omit it.

## Input and output

| Flag | Default | Meaning |
|---|---|---|
| `--image <path>` | *(one required)* | Image to recognize. Any format OpenCV can decode. Mutually exclusive with `--images-from`. |
| `--images-from <path>` | *(one required)* | Recognize every image listed in `<path>`, one path per line. `-` reads the list from stdin. See [Batch input](#batch-input). |
| `--json` | off | Emit machine-readable JSON and nothing else on stdout. See [the contract](#the-json-contract). |
| `--draw <path>` | *(unset)* | Write a copy of the image with every detected polygon outlined to `<path>`. Single-image mode only. |
| `--markdown <path>` | *(unset)* | Write the reconstructed markdown document to `<path>`. Implies `--word-boxes`. Single-image mode only. See [Markdown export](api/markdown.md). |
| `--word-boxes` | off | Also emit a polygon per word (per character for CJK). Adds `"words"` to each line. |

`--draw` colours boxes by recognition confidence: green at or above `0.5`, red
below. `0.5` is not a new number — it is the `--min-confidence` default, so a
red box is one the default config would have thrown away. You only ever see one
after lowering that filter, which is exactly when you are debugging.

## Batch input

`--images-from <path>` reads a list of image paths — one per line — and
recognizes every one of them in a single process. Pass `-` to read the list
from stdin.

```text
# invoices, Q3
scans/page-001.png
scans/page-002.png

# the rest of the batch
scans/page-003.png
```

Blank lines and lines whose first character is `#` are skipped, so a list file
can carry comments and survive being hand-edited. Everything else is taken
verbatim as a path — no globbing, no shell expansion, no trimming of anything
but the line ending. The list above therefore produces **three** results.

### Why this exists: the model load, not the spawn

Recall the framing at the top of this page: every wrapper drives this binary
over a subprocess. That is fine for one image and quietly catastrophic for a
document set. A 200-page job used to cost 200 process spawns **and 200 model
loads** — and it is the second number that hurts. Spawning a process is
milliseconds; opening three ONNX sessions, reading the recognizer's character
dictionary, and letting ORT build its graph optimizations is the dominant cost,
paid in full for every single page and thrown away immediately after.

`--images-from` constructs the `Engine` **once** and reuses it for every path in
the list. The per-image cost drops to what recognition actually costs. This is
the same win a long-lived `Engine` gives a C++ or Python caller, made available
to the subprocess-based wrappers that cannot hold one.

!!! tip "This is the flag to reach for before you reach for a GPU"

    If your throughput problem is "many pages", amortizing the model load is a
    larger and cheaper win than changing execution provider or model size. Fix
    the load pattern first, then benchmark.

### Why a list file rather than a glob or a repeatable flag

Three deliberate reasons, none of them about ergonomics:

- **No glob library, and no recursion policy to invent.** A `--images-dir`
  would immediately owe an answer to "does it recurse?", "which extensions?",
  "does it follow symlinks?", "what order?" — four decisions arboOCR would be
  making badly on your behalf. A list file has none: the caller already decided.
- **It composes with the tools that already do this well.** `find`, `ls`,
  `Get-ChildItem`, `git ls-files`, a database query, or a hand-written manifest
  all produce lines of text. Piping one into `--images-from -` costs nothing to
  learn.
- **No command-line length ceiling.** A repeatable `--image` flag runs into the
  operating system's argv limit — Windows caps a command line near 32k
  characters, which a few hundred realistic paths will exhaust. A file has no
  such bound, and the failure it avoids is the worst kind: it appears only once
  the batch gets large, in production.

### The JSON shape: an array, not an object

!!! warning "The JSON shape changes between single and batch mode"

    This is the one thing a wrapper author must not get wrong.

    | Mode | stdout with `--json` |
    |---|---|
    | `--image` | A **bare object** — `{"backend":…,"image":…,"lines":[…]}` |
    | `--images-from` | A **JSON array** of those same objects — `[{…},{…},{…}]` |

    The page objects are identical in both cases: same keys, same types, same
    [contract](#the-json-contract). Only the outer wrapping differs. Array order
    matches list order, and there is exactly one element per non-skipped input
    line, so you can zip the results back onto your inputs by index.

    A parser that assumes an object will fail on the first batch run, and one
    that assumes an array will fail on every single-image run. If your wrapper
    accepts both, branch on the flag you passed rather than sniffing the first
    byte.

Everything else about the `--json` contract holds: stdout carries pure JSON and
nothing else, diagnostics go to stderr, and the stream ends with one newline.

### One bad image does not abort the run

`recognize()` never throws. A path that does not exist, is not an image, or is
corrupt yields a page with `"lines":[]` and `elapsedMs` set — exactly as it does
in single-image mode — and the batch moves on to the next line. You get one
result object per input either way, so a typo on line 47 of a 200-line list
costs you line 47, not the other 199.

That means **you cannot detect a bad input from the exit code alone**. If a
missing file must be an error in your pipeline, check the paths before you write
the list, or check for empty `lines` per element afterwards.

### Exit codes in batch mode

The batch run distinguishes three outcomes:

| Code | Meaning |
|---|---|
| `0` | Every image produced at least one line. |
| `1` | The run completed, but one or more images produced no text. |
| `2` | Genuine failure — the engine could not be built, or the list could not be read. |

`1` is the interesting one: it is a *partial* result, not a failed run. The
output is complete and parseable; some pages were simply blank, unreadable, or
below `--min-confidence`. Treat `1` as "look at the results" and `2` as "nothing
usable came out". Only `2` means retrying with different inputs is pointless.

### Overlay and markdown output are rejected in batch mode

`--draw` and `--markdown` both take a single output path. Against a list of N images they would write the
same file N times and leave you with the last one, which is worse than useless
because it looks like it worked. Combining either with `--images-from` is
therefore a usage error, not a silent overwrite.

!!! note "The upgrade path is an output directory, not a mangled filename"

    If per-image overlays or markdown for a batch turn out to be wanted, the fix
    is an explicit output-directory flag (`--draw-dir`, `--markdown-dir`) that
    derives one file per input — not a template string, and not
    `overlay.png` silently becoming `overlay-001.png`. Nothing is stopping that
    from being added; it just has not been needed yet. Until then, loop the
    single-image form for the handful of pages you actually want to inspect —
    debugging is not the throughput path.

### Batch examples

=== "A list file"

    ```bash
    arboocr_demo --images-from pages.txt --models-dir models --json > pages.json
    ```

    `pages.json` holds one array element per line of `pages.txt`, in order.

=== "A pipeline"

    ```bash
    ls *.png | arboocr_demo --images-from - --models-dir models --json
    ```

    `-` reads the list from stdin, so anything that emits one path per line
    feeds the batch directly. On Windows PowerShell:

    ```powershell
    Get-ChildItem *.png | Select-Object -ExpandProperty FullName |
      arboocr_demo --images-from - --models-dir models --json
    ```

=== "A recursive scan"

    ```bash
    find scans/ -type f -name '*.jpg' | sort |
      arboocr_demo --images-from - --models-dir models --json > out.json
    ```

    The recursion policy, the extension filter and the ordering are all yours —
    which is the entire point of taking a list instead of a directory.

=== "Human-readable batch"

    ```bash
    arboocr_demo --images-from pages.txt --models-dir models
    ```

    Without `--json` you get the same per-image block as single-image mode,
    one after another. Useful for eyeballing a set; parse the JSON form instead.

## Model selection

| Flag | Default | Meaning |
|---|---|---|
| `--models-dir <dir>` | `models` | Directory holding the ONNX files. |
| `--ocr-version <str>` | `PP-OCRv6` | Model family; used as the filename prefix. |
| `--model-type <str>` | `small` | Recognizer size — `tiny`, `small`, or `medium`. |
| `--det-model <path>` | *(none)* | Override the resolved detector path. |
| `--cls-model <path>` | *(none)* | Override the resolved classifier path. |
| `--rec-model <path>` | *(none)* | Override the resolved recognizer path. |
| `--dict <path>` | *(none)* | Override the character dictionary path. |
| `--no-download` | off | Never fetch missing models — fail instead. Same effect as `ARBOOCR_OFFLINE=1`. |
| `--models-url <url>` | *(none)* | Directory URL to fetch missing models from. Empty uses the pinned default release. |
| `--download-models` | off | Fetch the models for `--ocr-version`/`--model-type` into the cache and exit without recognizing anything. See [Prewarming the cache](#prewarming-the-cache-download-models). |

`--no-download` and `--download-models` are presence switches like every other
boolean flag on this page — write `--no-download`, not `--no-download=true`.

Unless overridden, paths are assembled from the three values above:

```text
<models-dir>/<ocr-version>_det.onnx
<models-dir>/<ocr-version>_cls.onnx
<models-dir>/<ocr-version>_rec_<model-type>.onnx
<models-dir>/<ocr-version>_rec_<model-type>_dict.txt
```

!!! warning "`--model-type` sizes the recognizer, not the detector"

    The detector path is always `<ocr-version>_det.onnx` — the `_type` suffix
    is only applied to the recognizer and its dictionary. If your models
    directory contains `PP-OCRv6_det_tiny.onnx`, nothing loads it unless you
    pass `--det-model` explicitly. See [Model sizes](models/sizes.md).

Two more resolution details worth knowing before you debug a load failure. The
classifier is only opened when `--angle` is set, so a missing `_cls.onnx` is
harmless otherwise. And `--dict` is a **fallback**: the recognizer first tries
to read its character set from the ONNX file's own metadata and only touches
the `.txt` when that is absent. Run with `--log-level debug` to see which of
the two paths was taken.

### Missing models fetch themselves

A model file that is not on disk is no longer an immediate failure. Before the
first session opens, the binary resolves all four paths, downloads whatever is
missing from a pinned release, verifies each file against a SHA-256 compiled
into the binary, and writes it atomically into a per-user
[cache directory](#where-the-cache-lives). The atomic write earns its keep the
first time a download is killed halfway: without it you are left with a file
that *exists* and does not load, which every later run treats as a cache hit
and fails on.

Resolution is per file, not per run, and it stops at the first hit:

| Precedence | Source | When |
|---|---|---|
| 1 | `--det-model`, `--cls-model`, `--rec-model`, `--dict` | An explicitly given path is used as-is and is **never** substituted by a download. |
| 2 | `--models-dir` | An existing, non-empty file there wins — a populated models directory means zero network access. |
| 3 | The cache | Fetched from `--models-url` (or the default release), SHA-256 verified, and reused by every later run. |

!!! danger "A fine-tuned model is never silently swapped for a stock one"

    Rule 1 is the load-bearing one. Point `--rec-model` at your own weights and
    misspell the path, and arboOCR does **not** quietly download the stock
    recognizer and carry on emitting plausible-looking text from a model you
    did not choose. You get the model-load failure, naming the file you
    actually asked for. Silent substitution would be undetectable from the
    output — the results would just be quietly wrong.

Both resolution details above carry over to fetching. The classifier is only
downloaded when `--angle` is set, and the dictionary is best-effort for the
same reason it is a fallback on disk — the recognizer usually carries its own
charset, and repos hosting such models often publish no `.txt` at all. Neither
absence is a failure.

### Prewarming the cache: `--download-models`

`--download-models` fetches the models for the current `--ocr-version` and
`--model-type`, then exits without recognizing anything. It exists so the
network access happens where you can see it — a Docker build stage, a CI setup
step, a provisioning script — instead of inside the first request that a real
user is waiting on.

It prints one line per file, in det, cls, rec, dict order:

```text
$ arboocr_demo --download-models --ocr-version PP-OCRv6 --model-type small
ok      C:\Users\you\AppData\Local\arboOCR\models\models-v1\PP-OCRv6_det.onnx
missing C:\Users\you\AppData\Local\arboOCR\models\models-v1\PP-OCRv6_cls.onnx
ok      C:\Users\you\AppData\Local\arboOCR\models\models-v1\PP-OCRv6_rec_small.onnx
missing C:\Users\you\AppData\Local\arboOCR\models\models-v1\PP-OCRv6_rec_small_dict.txt
```

Exit `0` when the detector and the recognizer are both present, `2` otherwise.
Both `missing` lines above are expected and neither affects the exit code: no
`--angle`, so no classifier was wanted, and this recognizer embeds its own
charset. Keying the result on det and rec rather than on four clean `ok`s is
what keeps a CI gate from failing over a dictionary nobody needs.

!!! warning "`--download-models` with `--no-download` is a usage error"

    Asking for a download and forbidding downloads in the same command line has
    no sensible interpretation, so it is rejected with exit `1` rather than
    resolved in favour of one of them. If you want a prewarm step that is a
    no-op offline, branch on the environment yourself — do not pass both and
    hope.

=== ":material-docker: A Docker build stage"

    ```dockerfile
    ENV ARBOOCR_CACHE_DIR=/opt/arboocr/models
    RUN arboocr_demo --download-models --model-type medium
    ENV ARBOOCR_OFFLINE=1
    ```

    The fetch happens once, in a cached layer. Set `ARBOOCR_CACHE_DIR`
    explicitly here: the default cache lives under the *user's* home, so a
    build stage running as `root` and a service running as anyone else fill and
    read two different directories — and the symptom is a container that
    downloads models on every cold start. `ARBOOCR_OFFLINE=1` at runtime turns
    a cache miss into a loud failure instead of a surprise egress from
    production.

=== ":material-github: A CI setup step"

    ```bash
    arboocr_demo --download-models --model-type small || exit 1
    arboocr_demo --images-from pages.txt --no-download --json > out.json
    ```

    The first line is the only one allowed to touch the network, and it fails
    the job loudly if the release is unreachable. The second runs against the
    cache with fetching switched off, so a run that would silently re-download
    on a cache miss fails instead — which is how you find out your cache key is
    wrong.

## Environment

Three variables cover what a flag cannot: a build agent that must never reach
the network, an image with a baked-in cache, and a site that mirrors the
release internally.

| Variable | Effect |
|---|---|
| `ARBOOCR_OFFLINE=1` | Forbid all model fetching, process-wide. Equivalent to `--no-download`. |
| `ARBOOCR_CACHE_DIR` | Override the cache root. The `models-v1` tag segment is still appended. |
| `ARBOOCR_MODELS_URL` | Override the default base URL — point a fleet at an internal mirror. |

`ARBOOCR_OFFLINE` and `--no-download` are the same switch reached two ways, and
either alone is enough: the flag is for one command, the variable for every
process in an environment you do not want editing every call site of.
`ARBOOCR_MODELS_URL` changes what an empty `--models-url` resolves to, so a
mirror can be configured once for a machine rather than threaded through every
invocation.

!!! info "What the default URL points at"

    `https://github.com/ARBO-TEAM/arbo-ocr-models/releases/download/models-v1/`
    — release assets under an immutable tag, so the bytes behind a given
    filename cannot change under a deployment that already pinned this version.
    Every stock file is checked against a SHA-256 baked into the binary, so a
    truncated transfer, a captive-portal login page served with a `200`, or a
    tampered mirror all fail the download rather than landing on disk and
    failing later as a confusing model-load error.

### Where the cache lives

| Platform | Path |
|---|---|
| Windows | `%LOCALAPPDATA%\arboOCR\models\models-v1` |
| macOS | `~/Library/Caches/arboOCR/models/models-v1` |
| Linux | `$XDG_CACHE_HOME/arboOCR/models/models-v1`, else `~/.cache/arboOCR/models/models-v1` |

The trailing `models-v1` is the release tag, and it is not decoration. Scoping
the directory by tag is what makes a future `models-v2` structurally incapable
of reading a `models-v1` file: a re-trained weight shipped under the same
filename lands in a different directory instead of registering as a cache hit
and being loaded for the rest of that machine's life. `ARBOOCR_CACHE_DIR` moves
the root only — the tag segment is still appended underneath whatever you set.

!!! tip "The cache is disposable"

    Nothing in it is authoritative; every file can be re-fetched and is
    verified on arrival. Deleting the directory is a valid first step when a
    model behaves oddly, and costs one download. That is also why downloads
    land in the cache rather than in `--models-dir`: the directory you curated
    is not somewhere an automatic fetch gets to add files.

## Detection tuning

These map one-to-one onto `EngineConfig`; the reasoning behind each default
lives in [Accuracy defaults](models/accuracy-defaults.md).

| Flag | Default | Meaning |
|---|---|---|
| `--det-limit-side-len <int>` | `960` | Longest image side for the detection resize. Raise for dense small text, at a latency cost. |
| `--det-thresh <float>` | `0.3` | Probability-map binarization threshold. |
| `--det-box-thresh <float>` | `0.5` | Minimum mean score for a detected box to survive. Lower it to recover faint boxes. |
| `--det-unclip-ratio <float>` | `1.6` | Box expansion applied after binarization. Raise it when crops clip ascenders/descenders. |
| `--split-overmerged` | off | Split wide detection boxes that fused two side-by-side fields, using an ink-gap heuristic. |

`--split-overmerged` is the one to reach for on forms and tables, where a
`label` and its `value` sit on the same baseline and the detector happily
draws one box around both.

## Recognition tuning

| Flag | Default | Meaning |
|---|---|---|
| `--rec-batch-num <int>` | `6` | Crops per recognition inference call. Clamped to `>= 1`. |
| `--min-confidence <float>` | `0.5` | Drop lines below this recognition confidence. `0` disables filtering entirely. |

!!! tip "`--min-confidence 0` is a diagnostic, not a setting"

    When text is missing from the output, run once with `--min-confidence 0
    --draw out.png`. If the line appears in red, the detector found it and the
    filter discarded it — a recognition problem. If it does not appear at all,
    the detector never saw it, and no recognizer setting will help; go tune
    detection or reach for [`--clahe`](models/clahe.md).

## Backend

| Flag | Default | Meaning |
|---|---|---|
| `--cuda` | off | Request the CUDA execution provider. |
| `--tensorrt` | off | Request TensorRT. Implies CUDA as the fallback tier. |
| `--fp16` | **on** | TensorRT FP16. Only consulted with `--tensorrt`. Disable with `--fp16=false`. |
| `--trt-cache-dir <dir>` | `models/trt_engines` | Where TensorRT caches built engines. Only used with `--tensorrt`. |
| `--angle` | off | Enable orientation classification (0°/180°). Loads `_cls.onnx`. |
| `--clahe` | off | CLAHE contrast enhancement before detection — for faded/low-contrast scans. |

Requests are requests, not guarantees. The engine probes
`Ort::GetAvailableProviders()` and degrades TensorRT → CUDA → CPU silently, so
`--tensorrt` on a machine without it still runs. The `"backend"` field in
`--json` output (and the `Backend:` line otherwise) reports what was *actually*
selected — check it before you trust a benchmark.

!!! warning "Changing `--fp16` or `--rec-batch-num` invalidates cached TRT engines"

    TensorRT engines under `--trt-cache-dir` are built for a specific precision
    and a specific recognizer batch size — `--rec-batch-num` is applied *before*
    model load precisely so the TRT optimization profile matches the runtime
    batch shape. The cache directory is not keyed on either value, so flipping
    `--fp16=false` or moving `--rec-batch-num` from `6` to `12` against a
    populated cache makes TensorRT load a mismatched engine. Clear the directory
    or point `--trt-cache-dir` at a separate path per configuration.

## Diagnostics

| Flag | Default | Meaning |
|---|---|---|
| `--log-level <level>` | *(silent)* | Log engine events to stderr: `debug`, `info`, `warn`, or `error`. |
| `-h`, `--help` | — | Print usage and exit. |

!!! note "Logging is opt-in — this is a behaviour change"

    Without `--log-level` the library installs **no log callback at all** and
    writes nothing to stderr. It previously always logged. If you had a wrapper
    or a shell pipeline reading engine diagnostics off stderr, it now reads
    nothing until you pass the flag. An unrecognized level (`--log-level
    verbose`) is a usage error, not a silent fallback: it prints the accepted
    set and exits `1`.

```text
$ arboocr_demo --image page.jpg --models-dir models --log-level debug
[arboOCR][INFO] Engine backend: cpu
[arboOCR][DEBUG] Model paths det=models\PP-OCRv6_det.onnx cls=models\PP-OCRv6_cls.onnx rec=models\PP-OCRv6_rec_small.onnx dict=models\PP-OCRv6_rec_small_dict.txt
[arboOCR][DEBUG] Recognizer keys: loaded from model metadata
[arboOCR][DEBUG] recognize: 3 lines
```

!!! tip "arboOCR is silent; ONNX Runtime is not always"

    "Silent by default" is a promise about *arboOCR's* logging. ONNX Runtime
    writes its own diagnostics to stderr independently — some vcpkg Windows
    builds emit a wall of `Schema error: Trying to register schema ...` lines
    on every start, which are harmless duplicate-registration warnings from
    onnx itself. A wrapper that treats "any stderr output" as failure will
    misfire on those. Key on the **exit code** instead.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Success. |
| `1` | Usage or parse error, or (default output mode) no text found. |
| `2` | Model load failure — including a fetch that was attempted and failed — or an unexpected recognition error. |

Wrappers should treat any non-zero code as failure. `2` is the one worth
special-casing: it means the engine could not be *built* — a missing, corrupt,
or unreadable ONNX file, or a missing one that could not be downloaded — or
that inference itself threw. On `2` the binary prints the four resolved model
paths to stderr, plus whether auto-download was on or off and the cache
directory it resolved to, so you can see both which file it went looking for
and where it was entitled to look.

`1` covers a wider range: an unrecognized flag, no input flag at all, `--image`
and `--images-from` together, `--draw` or `--markdown` alongside
`--images-from`, `--download-models` alongside `--no-download`, a bad
`--log-level` value, and — in the default human-readable mode — a page that
produced zero lines.

!!! warning "`2` is no longer always permanent"

    The old rule was that retrying a `2` is pointless, because a missing file
    stays missing. With auto-download on, `2` also covers "the fetch failed",
    and a fetch fails for reasons that go away by themselves: a proxy, a rate
    limit, a runner with no egress that hour. The stderr block tells you which
    case you are in — if auto-download was `off`, the old rule still holds and
    a retry is wasted; if it was `on`, read the download error before you
    decide. The way to not have this failure mode at request time at all is to
    move the fetch into your build with
    [`--download-models`](#prewarming-the-cache-download-models).

!!! note "`--download-models` has only two outcomes"

    That mode recognizes nothing, so the codes mean something narrower: `0`
    when the detector and recognizer are both on disk when it finishes, `2`
    when either is not. A `missing` line for the classifier or the dictionary
    does not change the result. See
    [Prewarming the cache](#prewarming-the-cache-download-models).

!!! note "Batch mode reads `1` as partial, not failed"

    With `--images-from`, `1` means the run finished and produced complete
    output but at least one image yielded no text. Unlike the single-image
    `--json` case below, that holds in both output modes: the batch exit code
    reports the *set*, not the last page. See
    [Exit codes in batch mode](#exit-codes-in-batch-mode).

!!! warning "`--json` reports an empty page as success"

    The empty-page case is the one asymmetry between the two output modes. In
    `--json` mode a page with no text exits **`0`** with `"lines":[]`, because
    an empty result set is a valid result, and wrappers need to distinguish "the
    scan is blank" from "the binary broke". The same is true of an unreadable or
    missing image file — `recognize()` never throws, so it degrades to empty
    lines rather than an error. If your wrapper must reject empty pages, check
    `len(lines) == 0` yourself; do not expect a non-zero exit.

## The JSON contract

With `--json`, stdout carries **pure JSON and nothing else** — one compact
object, one trailing newline. This is load-bearing: wrappers pipe the entire
stream into a JSON parser without pre-filtering it, so a single stray
`printf` would break all four languages at once. Everything that is not the
result — engine logs, `--draw` confirmations, `--draw` failures — goes to
stderr.

The object below is the **single-image** shape. With
[`--images-from`](#the-json-shape-an-array-not-an-object) the same object
appears as one element of a JSON array; every key documented here is unchanged.

```json
{"backend":"cpu","image":"page.jpg","elapsedMs":66.3257,"lines":[{"text":"INVOICE","score":0.999803,"detScore":0.857679,"polygon":[{"x":43,"y":27},{"x":285,"y":30},{"x":284,"y":91},{"x":43,"y":89}]}]}
```

The same payload, re-indented for reading (the wire format is always the single
line above):

```json
{
  "backend": "cpu",
  "image": "page.jpg",
  "elapsedMs": 66.3257,
  "lines": [
    {
      "text": "INVOICE",
      "score": 0.999803,
      "detScore": 0.857679,
      "polygon": [
        {"x": 43, "y": 27}, {"x": 285, "y": 30},
        {"x": 284, "y": 91}, {"x": 43, "y": 89}
      ]
    }
  ]
}
```

| Key | Type | Notes |
|---|---|---|
| `backend` | string | `cpu`, `cuda`, or `tensorrt` — what was actually selected, not what was requested. |
| `image` | string | The **file name** of the input, not the path you passed. |
| `elapsedMs` | number | Wall-clock pipeline time. Set even when `lines` is empty. |
| `lines[].text` | string | Decoded text. UTF-8; wrappers must decode it as such. |
| `lines[].score` | number | Mean CTC character confidence. |
| `lines[].detScore` | number | Detector box score for this polygon. |
| `lines[].polygon` | array | Four `{x, y}` points in source-image coordinates. |
| `lines[].words` | array | **Only present with `--word-boxes`**, and only on lines that produced words. |

`lines` is sorted top→bottom, left→right by centroid — not raw detector order.
Floats are serialized at 6 significant digits, so `0.9f` reads as `0.9` rather
than binary noise.

The `"words"` omission is deliberate rather than an oversight: leaving the key
out entirely when empty means the JSON shape is byte-for-byte unchanged for
every caller that never asks for word boxes, so adding the feature broke no
existing wrapper. Treat `words` as optional in your parser.

```json
"words":[{"text":"PT.","score":0.999931,"polygon":[{"x":58.2754,"y":105.116},{"x":110.101,"y":105.464},{"x":110.101,"y":157.464},{"x":58.2754,"y":157.116}]},{"text":"Angin","score":0.99997,"polygon":[{"x":127.377,"y":105.58},{"x":222.391,"y":106.217},{"x":222.391,"y":158.217},{"x":127.377,"y":157.58}]}]
```

!!! note "Word polygons are alignment-derived, not supervised"

    They come from CTC timestep alignment, a by-product of recognition. CTC
    peaks partway through a glyph, so boxes run roughly half a character wide of
    true extents and lag consistently right. Good enough for highlighting,
    search hit-marking and reading-order reconstruction; not good enough to crop
    a glyph from.

### `--draw` composes with `--json`

They are not exclusive. The overlay is written *before* the JSON is emitted, and
its confirmation and error messages go to stderr, so stdout stays a single
parseable object:

```bash
arboocr_demo --image page.jpg --models-dir models --json --draw overlay.png > page.json
```

`page.json` is valid JSON; `overlay.png` is on disk. This is the shape you want
in a debugging pipeline that also needs the structured result.

This composition is single-image only — `--draw` with `--images-from` is
[rejected](#overlay-and-markdown-output-are-rejected-in-batch-mode), because one
output path cannot hold N overlays.

## Examples

=== "Simplest run"

    ```bash
    arboocr_demo --image page.jpg --models-dir models
    ```

    ```text
    Backend: cpu
    Image: page.jpg
    Lines: 3 (63.9146 ms)
      [0] "INVOICE" (score=0.999803 detScore=0.857679) poly=[(43,27), (285,30), (284,91), (43,89)]
      [1] "PT. Angin Sepoi" (score=0.997808 detScore=0.846785) poly=[(41,105), (339,107), (339,159), (41,157)]
      [2] "Total 1250000" (score=0.999852 detScore=0.854063) poly=[(44,177), (303,174), (303,223), (44,226)]
    ```

=== "Driving it from a wrapper"

    ```bash
    arboocr_demo --image page.jpg --models-dir models --json --word-boxes
    ```

    One compact JSON object on stdout, exit `0`. This is the invocation every
    wrapper builds internally — see [Wrappers](wrappers/index.md).

=== "A whole folder"

    ```bash
    ls *.png | arboocr_demo --images-from - --models-dir models --json
    ```

    One process, one model load, a JSON **array** on stdout. This is the
    difference between a 200-page job that loads the models once and one that
    loads them 200 times — see [Batch input](#batch-input).

=== "Faded receipts"

    ```bash
    arboocr_demo --image receipt.jpg --models-dir models \
      --clahe --min-confidence 0 --split-overmerged
    ```

    `--clahe` rescues boxes the detector would otherwise miss on thermal-printer
    output; `--min-confidence 0` keeps every line so you can judge for yourself
    what the filter would have eaten; `--split-overmerged` separates the
    item-name and price columns that a receipt's tight layout fuses into one
    box.

=== "Debugging"

    ```bash
    arboocr_demo --image page.jpg --models-dir models \
      --log-level debug --min-confidence 0 --draw boxes.png
    ```

    stderr tells you which model files were opened and which backend loaded;
    `boxes.png` tells you what the detector actually saw. Red boxes are the
    lines the default `0.5` filter would have dropped.

=== "GPU"

    ```bash
    arboocr_demo --image page.jpg --models-dir models \
      --tensorrt --trt-cache-dir cache/trt-fp16-b6 --rec-batch-num 6
    ```

    First run pays the engine build; later runs load from the cache. Note the
    cache path encodes the precision and batch size — see the
    [warning above](#backend).

---

Next: [Wrappers](wrappers/index.md) to call this from Python, Go, Rust or PHP,
[Accuracy defaults](models/accuracy-defaults.md) for what the tuning flags
actually do, or the [API reference](api/index.md) if you would rather link the
library than spawn it.
