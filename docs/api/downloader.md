---
title: Model downloader
---

# Model downloader

arboOCR still does not *bundle* model files — but it now knows where to get
them. `model_downloader.hpp` is the layer underneath that: the default source,
the integrity baseline, and the fetch functions the `Engine` calls on your
behalf when a model is missing.

```cpp
struct DownloadResult {
    bool ok = false;
    std::string errorMessage;
    size_t bytesWritten = 0;
};

const char* defaultModelsTag();       // "models-v1"
std::string defaultModelsBaseUrl();   // release assets pinned to that tag
std::string defaultModelsCacheDir();  // per-platform cache dir, scoped by tag

std::string sha256File(const std::string& path);
std::string knownSha256(const std::string& fileName);  // "" for names we do not ship

DownloadResult downloadFile(const std::string& url, const std::string& destPath,
                            const std::string& expectedSha256 = "");

std::vector<std::string> ocrModelFileNames(
    const std::string& ocrVersion,
    const std::string& modelType);

std::vector<DownloadResult> downloadOcrModels(
    const std::string& baseUrl, const std::string& ocrVersion,
    const std::string& modelType, const std::string& modelsDir);
```

Typical use is now no use at all: construct an `Engine` and the files arrive.
When you want to drive the fetch yourself — a provisioning step, a warm image
layer, a prefetch before you drop network access:

```cpp
arbo::ocr::downloadOcrModels(
    "",  // empty baseUrl — use the default, pinned source
    "PP-OCRv6", "medium", "models");
```

!!! info "There is a default URL now — and here is what had to be true first"
    This page used to say there never would be one, and that was the right
    call while it stood. The objection had three parts, and a default is only
    safe when all three are answered:

    | The old objection | What answers it |
    |---|---|
    | A hardcoded URL **rots** | The default resolves to an immutable release tag, `models-v1`. That makes it a version rather than a moving target — assets under a published tag do not change underneath you, and a future `models-v2` gets its own URL *and* its own cache directory instead of overwriting this one. |
    | You do not control **integrity** | Every stock file name carries a SHA-256 compiled into the binary. A host that serves different bytes — a stale mirror, a captive-portal HTML page, a truncated transfer — is rejected rather than handed to ONNX Runtime. |
    | You do not control the **licence question** | The weights live in [arbo-ocr-models](https://github.com/ARBO-TEAM/arbo-ocr-models) with a NOTICE stating their PP-OCR provenance; upstream PaddleOCR is Apache-2.0. See [License](../license.md). |

    The escape hatch that used to be the *only* option is still there, and it
    still comes first: your own `baseUrl`, your own `modelsDir`, your own
    explicit paths. A default is what happens when you express no preference —
    it never overrides one. See [Where each file comes from](#where-each-file-comes-from).

## Where each file comes from

`EngineConfig` gained two fields, and the `Engine` constructor now calls
`ensureOcrModels(config)` where it used to call `resolveModelPaths(config)`:

```cpp
bool autoDownload = true;    // false forbids all fetching
std::string modelsBaseUrl;   // empty — use defaultModelsBaseUrl()

ModelPaths ensureOcrModels(const EngineConfig& cfg);
```

Resolution is decided **per file**, not per engine, so a directory holding two
of the four models downloads only the two it lacks:

| Order | Source | When it wins |
|---|---|---|
| 1 | An explicit `detModelPath` / `clsModelPath` / `recModelPath` / `dictPath` | Whenever that field is non-empty. Returned as-is — **never** substituted by a download. |
| 2 | An existing, non-empty file under `cfg.modelsDir` | Whenever the file is already on disk. A fully populated `modelsDir` means zero network access. |
| 3 | A download into the cache directory | Only when 1 and 2 both miss, `autoDownload` is on, and `ARBOOCR_OFFLINE` is unset. |

!!! warning "An explicit path is never silently swapped for a stock one"
    Rule 1 is the one worth internalising. If `recModelPath` points at a
    fine-tuned recognizer and the path is wrong — a typo, a volume that did not
    mount, a build step that skipped — arboOCR does **not** fall back to
    downloading the stock model and running with it. You get the missing-model
    failure you would have got before the downloader existed. OCR output from
    the wrong weights looks entirely plausible, so a silent substitution is a
    bug you would ship for months without noticing; a loud failure is one you
    fix in a minute.

`cls` only enters the calculation when `useAngleCls` is on — there is no point
fetching a classifier you will never run. A dict that cannot be fetched stays
non-fatal for the reason it always was: the charset is frequently embedded in
the rec ONNX `character` metadata, so there is often nothing to fetch.

`ensureOcrModels` **never throws**. Anything it cannot obtain keeps its
resolved-but-missing path and is handed to the `Engine` unchanged, so an
air-gapped box with a cold cache behaves exactly as it did before any of this
existed — you get the same model-load failure at the same moment, not a new
exception type to catch.

## The cache directory

Downloads land in a platform-correct cache directory rather than next to your
binary or in the working directory, so multiple projects on one machine share
one copy and no build tree accumulates 73 MB of weights:

| Platform | Path |
|---|---|
| Windows | `%LOCALAPPDATA%\arboOCR\models\models-v1` |
| macOS | `~/Library/Caches/arboOCR/models/models-v1` |
| Linux | `$XDG_CACHE_HOME/arboOCR/models/models-v1`, else `~/.cache/arboOCR/models/models-v1` |

The trailing `models-v1` is `defaultModelsTag()`, and it is the reason the
cache is safe to keep forever. Weights are scoped by tag, so a future
`models-v2` cannot reuse a `models-v1` file that happens to share a filename —
the two live in different directories and never collide.

## Integrity: hashing and atomic writes

`knownSha256(fileName)` returns the baked-in digest for a stock file name, or
an empty string for a name arboOCR does not ship. `downloadOcrModels` looks it
up per file, which means **even a custom mirror gets integrity checking for
stock file names** — point `baseUrl` at your internal artifact store and a
corrupted or substituted `PP-OCRv6_det.onnx` is still caught. Custom and
fine-tuned names have no baseline to check against, so they stay unverified.

When `downloadFile` is given a non-empty `expectedSha256` it does not write the
destination directly. It writes a sibling `.<pid>.tmp`, hashes that, and only
on a match does it `std::filesystem::rename` the temp file onto the
destination — a replacing rename on both POSIX and Win32. Two consequences
worth relying on:

- A reader never observes a partial file. There is no window in which the path
  exists but holds half an ONNX model.
- Two processes racing the same model cannot corrupt each other. Each writes
  its own pid-suffixed temp file; the loser's rename simply replaces an
  already-correct file with an identical one.

!!! tip "A hash also repairs a file you already have"
    With a pinned hash, an **already-existing** destination is re-hashed and
    re-fetched if it does not match. That is what turns a truncated `.onnx`
    from a permanent problem into a self-healing one: a download killed by a
    dropped connection used to leave a file that existed, satisfied every
    "is it there?" check, and then failed to load forever. Now it is detected
    and replaced on the next run.

    Note that `sha256File` is exposed too, so you can verify a models directory
    you provisioned by other means without downloading anything.

SHA-256 is vendored in `model_downloader.cpp` — about seventy lines, not a
link against OpenSSL. Integrity checking costs you no new dependency and no
new build flag.

## What `downloadOcrModels` actually writes

**Four files** — the three ONNX models *and* the recognizer dictionary. They
are exactly the default flat filenames the `EngineConfig` path derivation
expects when `detModelPath`, `clsModelPath`, `recModelPath` and `dictPath` are
left empty, dropped into `modelsDir` with no subdirectories and no renaming
hooks.

`ocrModelFileNames(ocrVersion, modelType)` returns that list, and both the URL
(`baseUrl + name`) and the destination (`modelsDir/name`) are derived from it.
Call it when you want to check what will land on disk, pre-seed a cache, or
write a manifest, without performing a download:

```cpp
for (const auto& name : arbo::ocr::ocrModelFileNames("PP-OCRv6", "medium")) {
    std::cout << name << "\n";
}
// PP-OCRv6_det.onnx
// PP-OCRv6_cls.onnx
// PP-OCRv6_rec_medium.onnx
// PP-OCRv6_rec_medium_dict.txt
```

`downloadOcrModels` returns **one `DownloadResult` per name, always four
entries, always in that same order** — index 0 detector, 1 classifier, 2
recognizer, 3 dictionary. The vector length does not vary with success or
failure, so you can index it positionally. An empty `baseUrl` means the
default source; pass your own to override it.

!!! warning "The dictionary is best-effort — index 3 may be `ok=false` and still fine"
    A recognizer that embeds its charset in the ONNX `character` metadata key
    needs no dict file at all, and repos hosting such models often do not
    publish one. A 404 there is therefore **not** promoted to an error for the
    whole call: the fourth `DownloadResult` keeps the normal shape, comes back
    `ok=false`, and has its `errorMessage` suffixed to say the failure may be
    ignorable — that the dict is only needed when the rec model does not embed
    its charset in the ONNX `character` metadata key.

    You are not lied to (`ok` is still `false`) and you are not forced to guess
    what it means. Treat the first three failing as fatal; treat the fourth as
    a prompt to check whether your recognizer carries its own charset. If it
    does not, and the dict did not download, `Engine` will produce
    empty-lines pages rather than throwing.

!!! tip "Custom or fine-tuned models: use `downloadFile`"
    If your weights do not follow the default naming — a fine-tuned
    recognizer, a language-specific dictionary, an A/B variant you keep beside
    the stock file — call `downloadFile` once per artefact into explicit
    destination paths, then point the matching `EngineConfig::*ModelPath` /
    `dictPath` field at each one. `downloadOcrModels` has no escape hatch for
    that; it is a convenience for the stock layout only.

    Pass your own `expectedSha256` if you want the same verify-then-rename
    guarantee for those files — `knownSha256` has no entry for a name arboOCR
    does not publish.

## Turning it off

Auto-download is a default, not a policy. Five levers override it, listed in
the order you would reach for them — code first, then environment:

| Lever | Effect |
|---|---|
| `EngineConfig::autoDownload = false` | This engine fetches nothing; resolution stops after `modelsDir`. |
| `EngineConfig::modelsBaseUrl` | Fetch, but from your host instead of the default one. |
| `ARBOOCR_OFFLINE=1` | Process-wide: forbid all fetching, whatever the config says. |
| `ARBOOCR_CACHE_DIR` | Move the cache root — a shared read-only mount, a container volume, a build cache. |
| `ARBOOCR_MODELS_URL` | Override the base URL without touching code or config. |

The environment variables are the ones that matter in a container, because
they let an operator who cannot recompile still pin the behaviour. Setting
`ARBOOCR_OFFLINE=1` in a hardened image is the honest way to state "this box
does not talk to the internet" — you get a model-load failure at startup
instead of a network timeout on the first page.

The CLI mirrors all of it: `--no-download`, `--models-url <url>`, and
`--download-models` to prefetch and exit — the last is what you want in a
Dockerfile `RUN` step, so the weights bake into a layer instead of being
fetched on every container start. See [CLI](../cli.md#model-selection).

## Checking results

Both download functions return `DownloadResult`. Check it. A partial download
leaves you with a file that exists and does not load, and by the time `Engine`
sees it you get an empty-lines page rather than an exception — see the
[API Reference](index.md#arboocrengine).

Neither function throws: an unreachable host or an HTTP error comes back as
`ok=false` with a message. What `downloadFile` does with a destination that
already exists depends on whether a hash was pinned:

| `expectedSha256` | Existing, non-empty destination |
|---|---|
| Empty | **Skipped, unverified.** Returns `ok=true` with `bytesWritten = 0`. |
| Non-empty | **Re-hashed.** Matches, skipped the same way; does not match, re-fetched and replaced. |

So `bytesWritten == 0` still means "already had it", not "wrote nothing
useful", and re-running a provisioning step is still cheap and idempotent. The
difference is what "already had it" is allowed to mean: without a hash it means
*a file is there*, and with one it means *the right file is there*.

## See also

- [Models](../models/index.md) — the expected layout, and the three ways to get the files.
- [Languages](../models/languages.md) — which dictionary pairs with which recognizer.
- [License](../license.md) — what arboOCR's own licence covers, and what it does not.
