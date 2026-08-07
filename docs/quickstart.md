---
title: Quickstart
---

# Quickstart

Three ways in, shortest first. All of them need [model files](models/index.md) —
arboOCR does not bundle them.

## The CLI

If you just want to see it work, use the bundled demo binary — no code, no build
if you grab a [release](https://github.com/wafik/ArboOCR/releases):

```bash
./arboocr_demo --image page.jpg --models-dir models
```

```text
Backend: tensorrt
Image: page.jpg
Lines: 31 (402.053 ms)
  [0] "INVOICE" (score=0.845491)
  [1] "PT. Angin Sepoi" (score=0.862601)
  ...
```

## C++

```cpp
#include <arboOCR/engine.hpp>
#include <iostream>

int main() {
    arbo::ocr::EngineConfig cfg;
    cfg.modelsDir = "models";
    cfg.useTensorrt = true; // auto-falls back to CUDA, then CPU
    // cfg.recBatchNum = 8; // optional: crops per rec inference (default 6)

    arbo::ocr::Engine engine(cfg);
    std::cout << "Running on: " << engine.backend() << "\n";

    // Never throws: missing/unreadable images yield empty lines + elapsedMs.
    auto page = engine.recognize("page.jpg");
    if (page.lines.empty()) {
        std::cerr << "No text found (missing image or empty page)\n";
        return 1;
    }
    for (auto& line : page.lines) {
        std::cout << line.text << " (score=" << line.score << ")\n";
        // Polygon points — draw boxes with e.g. cv::polylines
        for (auto& pt : line.polygon)
            std::cout << "  (" << pt.x << "," << pt.y << ")";
        std::cout << "\n";
    }
    // Structured export for wrappers / web backends:
    // std::cout << arbo::ocr::toJson(page, /*pretty=*/true) << "\n";
}
```

!!! tip "`page.lines` is sorted, not raw detector order"

    Lines come back top→bottom, left→right by centroid — **not** in the order
    the detector emitted them. If you're migrating code that matched boxes by
    index, see [Accuracy defaults](models/accuracy-defaults.md).

Next: [build the library](build/index.md), or read the
[API reference](api/index.md).

## Python bindings (optional)

Engine facade via pybind11 — same native backends as C++. Off by default
(`ARBOOCR_BUILD_PYTHON=OFF`); enable when you have Python 3 development headers
and want `import arboocr`.

```powershell
cmake --preset windows-x64 -DARBOOCR_BUILD_PYTHON=ON
cmake --build build/windows-x64 --config Release --target _arboocr
$env:PYTHONPATH = "python"
# Windows: DLL path is auto-probed for build/windows-x64/vcpkg_installed/.../bin;
# override with $env:ARBOOCR_DLL_DIR = "...\vcpkg_installed\x64-windows\bin" if needed.
python -c "from arboocr import Engine, EngineConfig; print(EngineConfig().model_type)"
```

```python
from arboocr import Engine, EngineConfig, to_json

cfg = EngineConfig()
cfg.models_dir = "models"
cfg.use_tensorrt = False
engine = Engine(cfg)
page = engine.recognize("page.jpg")          # path
# page = engine.recognize(bgr_numpy_hxwx3)  # uint8 BGR only
for line in page.lines:
    print(line.text, line.score, [(p.x, p.y) for p in line.polygon])
print(to_json(page, pretty=True))
```

NumPy images must be **HxWx3 uint8 BGR** (OpenCV layout). No async, no low-level
`Detector`/`Recognizer` bindings, and no published wheels in v1 — build the
extension against your local `arboOCR` lib. Path overrides map to snake_case
(`rec_model_path`, etc.).

!!! note "Bindings vs. wrapper package"

    These pybind11 bindings are **not** the same thing as
    [`arbo-ocr-python`](wrappers/python.md). The bindings link the library
    in-process and need a C++ build; the wrapper package shells out to the
    prebuilt binary and needs no build at all. Pick the wrapper unless you need
    in-process calls.

## More examples

More usage patterns — including driving `Detector`/`Classifier`/`Recognizer`
directly instead of the `Engine` facade — live in
[`examples/`](https://github.com/wafik/ArboOCR/tree/main/examples), as small
buildable programs you can run immediately, not just read.
