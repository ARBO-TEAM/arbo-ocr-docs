---
title: Choosing a model size
---

# Choosing a model size

`EngineConfig::modelType` (`tiny` | `small` | `medium`) selects the
**recognizer** size only — the detector has no size variants, it's always the
single `PP-OCRv6_det.onnx` file.

!!! info "Defaults"

    `modelType` defaults to `small` (CPU). Prefer `tiny` for throughput,
    `medium` only when you have GPU/TensorRT headroom and have measured a real
    accuracy win on *your* data. Detection always loads
    `<ocrVersion>_det.onnx` unless you set `detModelPath` (there is no
    `modelType`-selected det file).

## The numbers

| modelType | File size | CPU latency (SROIE warm*) | Full-page sim* |
|---|---|---|---|
| `tiny` | 4.3 MB | ~165 ms | ~93.3% |
| `small` (**default**) | ~20 MB | ~530 ms | ~94.4% |
| `medium` | 73 MB | ~2120 ms (~4× small) | ~94.8% |

<sub>*Warm Python Engine on a Windows CPU, 5 SROIE receipt images (char similarity vs box GT). Jetson / TensorRT/CUDA change absolute ms; relative ranking of sizes is the point.</sub>

Read that table as a latency curve, not an accuracy curve. Going `small` →
`medium` costs roughly 4× the CPU time to buy well under half a point of
similarity. On CPU that is almost never the right trade; with GPU/TensorRT
headroom the calculus changes, but you should measure it rather than assume it.

## What this looks like on a real page

We tested this on a real scanned receipt: `tiny` misread "Melawai" as
"Melavwai" and "Atas nama" as "Atasnama"; `medium` got both right with no new
errors introduced.

That last clause is the part that matters. A bigger recognizer that fixes two
words while introducing three new ones is not an upgrade — the check is
*net* errors, not the errors you went looking for.

## Upsizing the detector is worse, not better

We also tried upsizing the *detector* instead
(`PP-OCRv6_det_medium.onnx`) and got a **worse** result — 6x slower and *more*
misreads, because the larger detector's box geometry didn't suit the recognizer
it was paired with.

!!! danger "Bigger detector ≠ better OCR"

    Detector and recognizer are a matched pair. A heavier detector produces
    different box geometry, and the recognizer was trained against crops from
    the default one. You pay 6× the detection time to hand the recognizer
    inputs it likes *less*.

**The rule:** if you want better accuracy, change `modelType` (the
recognizer), not the detector file.

If accuracy is your actual problem, read
[Accuracy defaults](accuracy-defaults.md) before you touch `modelType` at all —
on our smoke set the gap was layout, ordering, and detection scale, not
recognizer capacity.
