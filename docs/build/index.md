---
title: Build
---

# Build

arboOCR ships three CMake presets. Pick the one matching your target.

| Preset | Platform | Dependency source |
|---|---|---|
| `windows-x64` | Windows, MSVC | [vcpkg](https://vcpkg.io) |
| `linux-x64` | Linux x86_64 | vcpkg |
| `jetson` | aarch64 (Jetson/embedded) | apt + vendored onnxruntime |

The split is deliberate. The two desktop presets let vcpkg own the whole
dependency graph, which is the least-surprise option when you have the CPU
budget to build it. The `jetson` preset does not, because on an embedded board
that trade goes the other way — see [Jetson / aarch64](jetson.md).

## Prerequisites

Both desktop presets resolve dependencies through [vcpkg](https://vcpkg.io), so
you need a vcpkg checkout and `VCPKG_ROOT` pointing at it before you configure.
Everything else — OpenCV, ONNXRuntime, cURL, doctest, cxxopts — comes from
there.

Model files are **not part of the build**, and no longer a step you have to
remember. arboOCR does not bundle them, but it does fetch them: a freshly built
`arboocr_demo` pointed at an empty `modelsDir` downloads the stock weights it
needs on first run, checks each against a SHA-256 baked into the binary, and
caches them per platform.

Make it explicit on a machine that must not reach for the network mid-run — an
air-gapped runner, a hermetic image layer. `arboocr_demo --download-models`
prefetches and exits, and `ARBOOCR_OFFLINE=1` (or `--no-download`) turns a
missing model into an immediate error rather than a socket that hangs. See
[Models](../models/index.md).

## Desktop presets

=== ":material-microsoft-windows: Windows (vcpkg)"

    ```powershell
    $env:VCPKG_ROOT = "C:\vcpkg"
    cmake --preset windows-x64
    cmake --build build/windows-x64 --config Release
    ```

    !!! tip "DLL path"

        The Python wrapper auto-probes
        `build/windows-x64/vcpkg_installed/.../bin` for the runtime DLLs.
        Override it with
        `$env:ARBOOCR_DLL_DIR = "...\vcpkg_installed\x64-windows\bin"` if your
        layout differs.

=== ":material-linux: Linux x64 (vcpkg)"

    ```bash
    export VCPKG_ROOT=/path/to/vcpkg
    cmake --preset linux-x64
    cmake --build build/linux-x64
    ```

    Single-config generator, so there is no `--config Release` — the preset
    already pins the build type.

### Opt-in onnxruntime 1.28.0 prebuilt

Both desktop presets default to onnxruntime **1.23.2** from vcpkg, and that
default is unchanged — vcpkg has no 1.28 port, and CI stays on 1.23.2. The
opt-in exists for when the remaining latency lives inside `Session::Run`
itself rather than in pre/post-processing: oar-ocr v0.9.2 already ships ORT
1.28.0, and a DLL-swap experiment (1.23.2 headers + 1.28.0 runtime, 6/6
byte-identical outputs) showed no source changes are needed — only build
wiring. See [Benchmarks](../benchmarks.md#arboocr-vs-oar-ocr). (Upstream this
is the `perf/ort-1.28` branch — open PR at time of writing, so build from that
branch until it merges.)

```powershell
$env:VCPKG_ROOT = "C:\vcpkg"
cmake --preset windows-x64 -DARBOOCR_ORT_VERSION=1.28.0
cmake --build build/windows-x64 --config Release
```

```bash
export VCPKG_ROOT=/path/to/vcpkg
cmake --preset linux-x64 -DARBOOCR_ORT_VERSION=1.28.0
cmake --build build/linux-x64
```

At configure time CMake downloads the official Microsoft release archive for
your OS, verifies its SHA-256 against the hash pinned in
`cmake/ort_prebuilt.cmake`, extracts it under the build directory
(`_deps/ort-prebuilt`), and links its headers and runtime instead of the vcpkg
port. vcpkg still installs its own 1.23.2 copy in manifest mode, but nothing
references it.

Three limits. Windows/Linux x64 only — aarch64 keeps the [Jetson
flow](jetson.md), which already pairs 1.27.1 headers against a 1.28.x runtime
`.so`, and the two flags cannot combine (`ARBOOCR_ORT_VERSION=1.28.0` with
`ARBOOCR_USE_SYSTEM_DEPS=ON` is a configure error). Only `1.23.2` and `1.28.0`
are accepted; anything else fails at configure time. For an offline machine,
`-DARBOOCR_ORT_ARCHIVE_FILE=<path>` substitutes a local copy of the exact
archive (filename must match, hash still verified) for the download.

Consumers change slightly: an install built this way records the prebuilt
include/lib paths, so a downstream project still needs OpenCV and CURL from
vcpkg but no longer resolves an `onnxruntime` package. Packaging needs no
changes — the 1.28 archives ship `onnxruntime_providers_shared`, so the stock
runtime globs pick it up.

## Jetson / aarch64

vcpkg builds everything from source, which is impractical on a Jetson. That
path uses apt for OpenCV/cURL plus a fully self-contained vendored
onnxruntime, and it has enough moving parts to deserve
[its own page](jetson.md).

## Running the tests

```bash
cd build/<preset>
ctest                 # or: ./arboocr_tests
```

`ctest` is the wrapper; `./arboocr_tests` is the doctest binary underneath if
you want to pass doctest filters directly.

## Install

The build tree is not the delivery format. `cmake --install` copies the public
half of it — headers, static libs, the CLI, and a CMake package — to a prefix
you choose:

```bash
cmake --install build/<preset> --prefix /opt/arboocr
```

On Windows, a multi-config generator needs the config named explicitly:

```powershell
cmake --install build/windows-x64 --config Release --prefix C:\arboocr
```

What lands where, relative to the prefix (via `GNUInstallDirs`, so `lib` may be
`lib64` on some distros):

| Path | Contents |
|---|---|
| `include/arboOCR/` | Every public header — `engine.hpp`, `types.hpp`, `detector.hpp`, and the rest |
| `lib/` | `arboOCR` and `arboocr_clipper` static libraries |
| `bin/` | `arboocr_demo` |
| `lib/cmake/arboOCR/` | `arboOCRConfig.cmake`, `arboOCRConfigVersion.cmake`, `arboOCRTargets.cmake` |

Two things are deliberately *not* installed. `arboocr_tests` and the
`examples/` programs stay build-tree only — they are for developing arboOCR,
not for consuming it. `arboocr_demo` is installed, because the Python, Go, Rust,
PHP and JavaScript wrappers all spawn it as a subprocess; for those languages it is the
delivered artifact rather than a demo. See [Command line](../cli.md).

`arboocr_clipper` is installed alongside the main library for a linker reason,
not a usability one: a static library that links another static library leaves
those symbols unresolved for the consumer, so the vendored Clipper archive has
to travel with it. Its header stays private — no public arboOCR header includes
it, and you never name the target yourself.

### The `ARBOOCR_INSTALL` option

Install rules are generated only when arboOCR is the top-level project. Pull it
in with `add_subdirectory()` and they switch off, so vendoring arboOCR into your
build does not silently splice its headers and CLI into *your* `cmake --install`
output.

```bash
cmake -S . -B build -DARBOOCR_INSTALL=ON    # force on inside add_subdirectory
cmake -S . -B build -DARBOOCR_INSTALL=OFF   # force off for a standalone build
```

## Consuming arboOCR from another project

Point `CMAKE_PREFIX_PATH` at the prefix you installed to, and the package
behaves like any other CMake dependency:

```cmake
find_package(arboOCR CONFIG REQUIRED)

add_executable(myapp main.cpp)
target_link_libraries(myapp PRIVATE arboOCR::arboOCR)
```

```bash
cmake -S . -B build -DCMAKE_PREFIX_PATH=/opt/arboocr
```

The namespaced target carries the include directories and the whole transitive
link line, so there is no `arboOCR_INCLUDE_DIRS` variable to plumb by hand.

Version matching is `SameMinorVersion`, which is stricter than CMake's default
on purpose: pre-1.0, `0.1` and `0.2` are not interchangeable, so
`find_package(arboOCR 0.1 CONFIG REQUIRED)` will refuse a `0.2` install rather
than link you against a changed API.

!!! tip "`arboOCR::arboOCR` works either way"

    The same namespaced name is defined as an `ALIAS` in the build tree, so a
    project that vendors arboOCR via `add_subdirectory()` writes exactly the
    same `target_link_libraries` line as one that installs it. Switching
    between the two is a one-line change to how you acquire the sources, not a
    rewrite of your link rules.

!!! note "The consumer must be able to find OpenCV, onnxruntime and cURL"

    arboOCR is a **static** library and its dependencies leak through its public
    headers — `types.hpp` includes `<opencv2/core.hpp>`, and
    `detector.hpp`/`classifier.hpp`/`recognizer.hpp` include
    `<onnxruntime_cxx_api.h>`. So `arboOCRConfig.cmake` re-resolves them with
    `find_dependency(onnxruntime CONFIG)`, `find_dependency(OpenCV CONFIG)` and
    `find_dependency(CURL CONFIG)` before it loads the targets file. cURL is not
    in any public header but is linked `PUBLIC`, so the imported target still
    names `CURL::libcurl` and it must resolve too.

    All three therefore have to be discoverable on the **consumer's**
    `CMAKE_PREFIX_PATH`, not just on the machine that built arboOCR. A prefix
    path missing onnxruntime fails at configure time with a message that names
    the dependency and not arboOCR:

    ```text
    Could not find a package configuration file provided by "onnxruntime"
    ```

    With a vcpkg-built arboOCR, the fix is to give the consumer the same
    toolchain file (`-DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake`)
    rather than adding prefixes one at a time.

    `doctest` and `cxxopts` are *not* re-found. They are test- and CLI-only, so
    consumers of the library never need them.

!!! note "Jetson / `ARBOOCR_USE_SYSTEM_DEPS` installs differ"

    In that mode onnxruntime is a vendored `.so`, not a CMake package, so the
    config file uses the plain-module `find_dependency(OpenCV)` and
    `find_dependency(CURL)` and re-applies the recorded
    `ARBOOCR_ORT_INCLUDE_DIR` / `ARBOOCR_ORT_LIB_DIR` to the imported target
    itself. Those paths are baked in at build time — moving the vendored
    onnxruntime after installing breaks consumers. See
    [Jetson / aarch64](jetson.md).

## Backend selection at runtime

You do not pick the execution provider at build time. `Engine` auto-detects
**TensorRT**, then **CUDA**, then **CPU** via `Ort::GetAvailableProviders()`,
and `engine.backend()` reports what was actually selected at runtime.

```cpp
arbo::ocr::Engine engine(cfg);
std::cout << "Running on: " << engine.backend() << "\n";
```

!!! note "One binary, every machine — but log what it picked"

    This means the same binary degrades gracefully across machines: build once
    with GPU providers available, ship it, and it will fall back to CPU on a
    host that has none rather than failing to start. Always log
    `engine.backend()` when a benchmark result looks wrong — the usual cause is
    a silent fallback to CPU.

!!! warning "Ship `onnxruntime_providers_shared` or GPU never comes up"

    Graceful degradation cuts both ways: a package that is *missing* the
    provider library degrades to CPU just as quietly as a host with no GPU.
    Every arboOCR release archive before **v0.3.0** had exactly that bug — no
    `onnxruntime_providers_shared` was published, so `--cuda` and `--tensorrt`
    could not load from the release package on either platform. It stayed
    hidden because ONNX Runtime `dlopen`s the providers instead of linking
    them: nothing shows up in the import table, and `ldd` lists no missing
    dependency. v0.3.0 packages them on both platforms. If you assemble your
    own distribution, copy `onnxruntime_providers_shared.dll` (Windows) or
    `libonnxruntime_providers_shared.so` (Linux) next to the binary, and verify
    with `engine.backend()` rather than with a linker tool.
