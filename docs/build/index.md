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

Model files are a **separate step**. arboOCR does not bundle them, and a build
with no ONNX files in `modelsDir` will configure and compile cleanly while
recognizing nothing at runtime. Fetch them before your first run — see
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

## Backend selection at runtime

You do not pick the execution provider at build time. `Engine` auto-detects
**TensorRT**, then **CUDA**, then **CPU** via `Ort::GetAvailableProviders()`,
and `engine.backend()` reports what was actually selected at runtime.

```cpp
arbo::ocr::Engine engine(cfg);
std::cout << "Running on: " << engine.backend() << "\n";
```

!!! note

    This means the same binary degrades gracefully across machines: build once
    with GPU providers available, ship it, and it will fall back to CPU on a
    host that has none rather than failing to start. Always log
    `engine.backend()` when a benchmark result looks wrong — the usual cause is
    a silent fallback to CPU.
