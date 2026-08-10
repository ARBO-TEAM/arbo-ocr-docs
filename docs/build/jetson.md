---
title: Jetson / aarch64
---

# Jetson / aarch64

vcpkg builds everything from source, which is impractical on a Jetson. This
path uses apt for OpenCV/cURL and a **fully self-contained** vendored
onnxruntime — both the C++ headers *and* the CUDA/TensorRT-enabled runtime
`.so` live under `vendor/onnxruntime/`, so arboOCR doesn't depend on any other
project's Python environment once set up.

That last point is the design goal of the whole procedure. It is easy to get a
Jetson build working by linking against whatever `libonnxruntime.so` happens to
sit inside a system `site-packages`; it is not easy to keep that working after
someone upgrades a virtualenv. Copying the runtime into `vendor/` costs a few
hundred megabytes and buys a build that only depends on files you control.

## Verified configuration

The procedure below was last run end to end on this stack, with `backend`
confirming TensorRT actually loaded:

| | |
|---|---|
| Board | Jetson Orin Nano Super (8 GB, 6 cores) |
| JetPack / L4T | 7.2 / R39.2 |
| OS | Ubuntu 24.04 LTS |
| CUDA / TensorRT | 13.2 / 10.16.2 |
| onnxruntime | 1.28.0 (CUDA + TensorRT providers) |
| OpenCV / GCC / CMake | 4.8.0 (apt) / 13.3 / 3.28 |

!!! warning "The original Jetson Nano cannot run this"
    This page's stack needs JetPack 6 or newer. The original Jetson Nano
    (Maxwell, 2019) tops out at JetPack 4.6 — CUDA 10.2, TensorRT 8.2,
    Ubuntu 18.04 — so neither onnxruntime 1.28 nor TensorRT 10 is available
    there. "Jetson" on this site means an Orin-generation board.

## System packages

```bash
sudo apt install -y libopencv-dev libcurl4-openssl-dev doctest-dev libcxxopts-dev cmake build-essential
```

!!! bug "The package is `libcxxopts-dev`, not `cxxopts-dev`"
    Earlier revisions of this page said `cxxopts-dev`. No such package exists
    on Ubuntu 24.04 — `apt` fails with `E: Unable to locate package
    cxxopts-dev`, and because one bad name aborts the whole command, *nothing*
    in the line gets installed. The correct name is **`libcxxopts-dev`**
    (3.1.1 on 24.04), which ships `/usr/lib/cmake/cxxopts/cxxopts-config.cmake`
    and so satisfies `find_package(cxxopts CONFIG REQUIRED)`.

??? tip "If your distribution has no cxxopts package at all"
    cxxopts is a single header, so you can vendor it and skip the system
    package rather than hunting for a backport. Point `cxxopts_DIR` at a
    directory containing a two-line config:

    ```bash
    mkdir -p .deps/cxxopts/include
    curl -sSfL -o .deps/cxxopts/include/cxxopts.hpp \
      https://raw.githubusercontent.com/jarro2783/cxxopts/v3.2.0/include/cxxopts.hpp
    cat > .deps/cxxopts/cxxopts-config.cmake <<'CFG'
    add_library(cxxopts::cxxopts INTERFACE IMPORTED)
    set_target_properties(cxxopts::cxxopts PROPERTIES
        INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_CURRENT_LIST_DIR}/include")
    CFG
    ```

    Then add `-Dcxxopts_DIR=$PWD/.deps/cxxopts` to the configure step. This
    touches nothing outside the build tree, which is the right move on a shared
    or production device.

## Vendoring onnxruntime

onnxruntime arrives in two pieces from two different sources, because no single
distribution ships both what you need to compile and what you need to run.

!!! warning "The official aarch64 release tarball is CPU-only"

    Microsoft's `onnxruntime-linux-aarch64-*.tgz` release asset contains a
    CPU-only runtime. It is the right source for the **headers**, but if you
    link against its `.so` you will get a working build with no GPU
    acceleration at all — `engine.backend()` will report `cpu` and every
    TensorRT/CUDA benchmark on this site will be unreachable. Source the
    accelerated `.so` from a pip wheel (`pip install onnxruntime`) or from
    JetPack's preinstalled onnxruntime instead.

### 1. onnxruntime C++ headers

```bash
# 1. onnxruntime C++ headers (the pip wheel ships none). Use the latest
# release tarball whose headers are ABI-compatible with your runtime — e.g.
# v1.27.1 headers work against a 1.28.x runtime .so (the C API is stable):
mkdir -p vendor/onnxruntime && cd vendor/onnxruntime
curl -sL -o ort.tgz https://github.com/microsoft/onnxruntime/releases/download/v1.27.1/onnxruntime-linux-aarch64-1.27.1.tgz
tar xzf ort.tgz && rm ort.tgz && mv onnxruntime-linux-aarch64-1.27.1 dist
```

The header/runtime version mismatch is intentional and safe: onnxruntime's C
API is stable, so a 1.27.1 header set links correctly against a 1.28.x runtime.
You do not need to hunt for an exactly matching tarball.

### 2. onnxruntime runtime `.so` with CUDA/TensorRT

Continue in the same `vendor/onnxruntime` directory.

```bash
# 2. onnxruntime runtime .so with CUDA/TensorRT support. The official aarch64
# release tarball above is CPU-only — a pip wheel (`pip install onnxruntime`)
# or JetPack's preinstalled one ships the accelerated build instead. Copy it
# in (point SRC at wherever onnxruntime is installed on your machine):
SRC=/path/to/your/onnxruntime/capi
mkdir -p lib
cp "$SRC"/libonnxruntime.so.* "$SRC"/libonnxruntime_providers_*.so lib/
ln -sf $(basename "$SRC"/libonnxruntime.so.*.*.*) lib/libonnxruntime.so.1
ln -sf $(basename "$SRC"/libonnxruntime.so.*.*.*) lib/libonnxruntime.so
cd ../..
```

The `libonnxruntime_providers_*.so` glob is what carries CUDA and TensorRT. If
that copy silently matches nothing, you have a CPU-only source directory — go
back and point `SRC` at the accelerated install.

## Configure and build

```bash
cmake --preset jetson   # ARBOOCR_ORT_LIB_DIR defaults to vendor/onnxruntime/lib
cmake --build build/jetson -j$(nproc)
```

`ARBOOCR_ORT_LIB_DIR` is the single knob: it defaults to the
`vendor/onnxruntime/lib` you just populated, so if you keep onnxruntime
somewhere else you override that one variable rather than editing the preset.

!!! tip "On a device that is doing other work, do not use `-j$(nproc)`"
    A full build takes about 90 seconds on an Orin Nano at `-j3`, and half the
    cores is enough to stay off the critical path of anything else on the box:

    ```bash
    nice -n 10 cmake --build build/jetson -j3
    ```

    Peak extra RAM is well under 1 GB, so memory is not the constraint — CPU
    contention is. `nice` matters more than the job count: it lets any
    latency-sensitive process preempt the compiler outright.

!!! warning "`ARBOOCR_ORT_LIB_DIR` is burned into the binary as an RPATH"
    The build sets `INSTALL_RPATH` from this variable with
    `BUILD_WITH_INSTALL_RPATH`, so the resulting `arboocr_demo` hardcodes an
    **absolute** path to the onnxruntime directory. Check it with
    `readelf -d build/jetson/arboocr_demo | grep RUNPATH`.

    Two consequences worth knowing before you tidy anything up. Renaming or
    moving the source tree breaks the binary — use a symlink if you want a
    shorter path. And if you point `ARBOOCR_ORT_LIB_DIR` at another project's
    `vendor/onnxruntime`, you have made that project a permanent runtime
    dependency; give each checkout its own copy instead.

## Verify

```bash
cd build/jetson
ctest                 # or: ./arboocr_tests
```

Then confirm the accelerated provider actually loaded. `Engine` auto-detects
TensorRT, then CUDA, then CPU via `Ort::GetAvailableProviders()`, and
`engine.backend()` reports what was selected at runtime — if it says `cpu` on a
Jetson, step 2 did not take.

```bash
./arboocr_demo --image page.jpg --models-dir models --json | grep -o '"backend":"[a-z]*"'
```

```text
"backend":"tensorrt"
```

Read `backend` rather than trusting the flag. Requesting a provider is not the
same as getting one: if TensorRT cannot load, the engine falls back to CPU and
says nothing.

### What each backend costs

Same 31-line receipt, `tiny` recognizer, on the [verified
configuration](#verified-configuration) above. `engineMs` is the engine's own
reported inference time; wall clock adds process start and model load.

| Requested | `backend` | engineMs |
|---|---|---:|
| *(default)* | `cpu` | 968 ms |
| `--cuda` | `cuda` | 2426 ms |
| `--tensorrt` | `tensorrt` | **277–322 ms** |

TensorRT is roughly **3× faster than CPU**. CUDA being 2.5× *slower* than CPU
is not a misconfiguration — it is per-process execution-provider
initialisation, which a single small image cannot amortise. TensorRT avoids it
by loading a pre-built engine from `trtCacheDir` instead of compiling kernels
at startup.

!!! tip "One image per process hides most of the win"
    The first TensorRT run above was 479 ms and the warm ones 277–322 ms — the
    difference is engine cache loading, paid on every process start. If you are
    measuring throughput rather than one-shot latency, use `--images-from` so a
    single process handles the whole list with one Engine construction.
    Spawning the binary per image measures startup, not inference.

---

The [Benchmarks](../benchmarks.md) page is the right reference for what to
expect once the above is working.
