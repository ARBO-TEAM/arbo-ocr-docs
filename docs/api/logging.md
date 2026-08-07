---
title: Logging
---

# Logging

!!! note "The library does not print to stdout/stderr by default"
    Silence is the default and it is a guarantee, not an accident. A library
    that writes to `std::cout` from inside a request handler is a library you
    end up patching. Demos and examples may use `std::cout` / `std::cerr` for
    their own UI — that is not the core engine.

If you want output, you ask for it. Install a process-wide callback and route
into your stack (spdlog, glog, etc.):

=== "Demo helper"

    ```cpp
    #include <arboOCR/logging.hpp>

    // Demo helper:
    arbo::ocr::setLogCallback(arbo::ocr::makeStderrLogger());
    arbo::ocr::setMinLogLevel(arbo::ocr::LogLevel::Debug);
    ```

=== "Bridge to your logger"

    ```cpp
    #include <arboOCR/logging.hpp>

    // Or bridge to your logger:
    arbo::ocr::setLogCallback([](arbo::ocr::LogLevel level, const std::string& msg) {
        // spdlog::log(map(level), msg);
    });
    ```

`makeStderrLogger()` is the convenience path for demos and local debugging.
For anything that runs unattended, use the lambda form and map `LogLevel` onto
your own severity enum so OCR output lands in the same pipeline as everything
else.

## Levels

| Level | Emitted for |
|---|---|
| `Debug` | Verbose per-stage detail. Off unless you lower the minimum. |
| `Info` | **Default minimum.** Normal operational messages. |
| `Warn` | Recoverable problems — degraded config, fallbacks. |
| `Error` | Failures. Remember that `recognize()` still does not throw; an error here may be all you see. |

`setMinLogLevel()` sets the floor. It is `Info` unless you change it, so
`Debug` messages are dropped before they reach your callback — you do not pay
for formatting you never see.

!!! tip "Pair logging with `backend()`"
    Log `Engine::backend()` once at startup. A config that requested TensorRT
    but fell back to CPU is silent otherwise, and the symptom shows up much
    later as unexplained latency. See the
    [API Reference](index.md#arboocrengine).

## Callbacks that throw

Callbacks that throw are swallowed so a broken sink never crashes OCR. A
misconfigured log target, a full disk, a socket that went away — none of those
should take down a recognition pass. The exception is discarded, not
re-raised, and not logged (there is nowhere left to log it to).

The consequence is worth stating plainly: if your sink is broken you will see
nothing at all rather than a crash. Test the callback itself.
