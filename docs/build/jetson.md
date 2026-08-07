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

## System packages

```bash
sudo apt install -y libopencv-dev libcurl4-openssl-dev doctest-dev cxxopts-dev cmake build-essential
```

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
./arboocr_demo --image page.jpg --models-dir models
```

```text
Backend: tensorrt
```

---

Every number on the [Benchmarks](../benchmarks.md) page was measured on a
**Jetson Nano** with this build, so it is the right reference for what to
expect once the above is working.
