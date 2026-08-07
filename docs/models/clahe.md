---
title: Low-contrast documents (CLAHE)
---

# Low-contrast documents (CLAHE)

Faded thermal-printer receipts and other low-contrast scans can cause the
detector to miss text boxes entirely — **no amount of recognizer accuracy fixes
a box that was never detected.**

That framing is the whole point of this page. If a line never becomes a box, it
never reaches the recognizer, so upgrading `modelType` cannot help you. The
failure is upstream. CLAHE — Contrast Limited Adaptive Histogram Equalization —
attacks it upstream, by boosting local contrast in the image before detection
runs.

## Turning it on

Set `useClahe = true` to apply CLAHE to the full image before detection:

```cpp
arbo::ocr::EngineConfig cfg;
cfg.modelsDir = "models";
cfg.useClahe = true; // boosts local contrast before the detector runs
```

## Why it's off by default

Off by default — it adds per-image CPU cost and isn't universally beneficial
(it can amplify noise on already-good scans). This is a genuine trade-off, not
a universal win: on clean, well-exposed documents you pay latency to make the
detector's job slightly harder.

!!! tip "Enable it for a failure mode, not as a general accuracy knob"

    Turn `useClahe` on when you can point at pages where the detector is
    *missing text regions*, and verify it on your own data. If your problem is
    misread characters inside boxes that were detected fine, CLAHE is the wrong
    tool — see [Choosing a model size](sizes.md).

## What exactly gets enhanced

Only the detector's input image is enhanced. Recognizer crops are then taken
from that same enhanced image, for consistency — the recognizer sees pixels
from the same enhancement the detector saw, so box coordinates and crop
contents can't drift apart.

The enhancement itself only ever runs **once per page**, not once per detected
line. The cost is a fixed per-image overhead, not something that scales with
how much text is on the page.

CLAHE is one of the three opt-in switches (`useClahe`, `useAngleCls`,
`splitOvermerged`) that should stay off unless the failure mode matches — see
[Accuracy defaults](accuracy-defaults.md).
