---
date:
  created: 2026-08-07
authors:
  - arbo-team
categories:
  - Release notes
tags:
  - accuracy
  - models
description: >-
  Cold arbo small went from ~84% to ~94% char similarity on the SROIE receipt
  smoke set — and none of it came from a bigger recognizer.
---

# The accuracy cycle: the recognizer was never the problem

Cold `arbo` `small` went from **~84% to ~94%** char similarity on a five-image
SROIE receipt smoke set, landing within **~0.2 pts** of a TypeScript PaddleOCR
port measured on the same images. The interesting part is not the number — it
is that we got there without touching the recognizer size. Every point came
from layout, ordering, and detector scale.

<!-- more -->

## The diagnosis

The obvious move when full-page OCR reads at 84% is to reach for a bigger
model. We measured before we did that, and the measurement said no.

On the smoke set, **matched-line recognition was already ~97%**. When a line
was detected and matched to its ground-truth box, the recognizer usually got
the characters right. The ~13-point gap between that and the full-page score
was structural: lines emitted in detector order rather than reading order,
boxes over-merged at the wrong detector input scale, and missing spaces where
columnar receipt fields ran together.

That is a layout, ordering, and detector-scale problem — **not** a recognizer
capacity problem. Swapping in a heavier recognizer would have paid ~4x the CPU
latency to fix something the recognizer was not doing wrong.

## What actually changed

| Setting | Old | New default | Why |
|---|---|---|---|
| `modelType` | `medium` | **`small`** | Medium is ~4x slower on CPU for little sim gain on receipts |
| `detLimitSideLen` | `1536` | **`960`** | 1536 over-merged on dense receipts; 960 recovered ~2 pts |
| Reading order | detector order | **always sorted** (centroid y, then x) | Full-page string metrics and human reading both depend on order |
| CTC post-processing | greedy only | **gap to space + fullwidth to ASCII** | Spaces injected where CTC timesteps show wide gaps; fullwidth punctuation mapped to ASCII on non-CJK text |
| `minimumConfidence` | (none) | **`0.5`** | Drops low-confidence lines; `0` keeps every box |
| `splitOvermerged` | n/a | **`false`** | New, but **off** — see below |
| `LinePrediction.score` | unclear | **mean rec CTC confidence** | The detector box score moved to `detScore` |

Two of these deserve a note.

`splitOvermerged` is the flag we built and then defaulted **off**. It performs
an ink-gap split of wide fused detector boxes, which sounds like exactly the
right fix for over-merging — and on the smoke set aggressive splitting **lost
~1 pt**. It ships opt-in, for cases where the detector demonstrably fuses
side-by-side fields. Shipping it on by default would have quietly undone part
of the gain.

`LinePrediction.score` is a breaking change for anyone consuming it. It now
means mean recognizer CTC confidence; the detector's box score lives in the
new `detScore` field (`det_score` in JSON and Python). If you were treating
`score` as detection confidence, that read is now wrong.

## The numbers

Warm Python `Engine`, CPU, 5 SROIE receipts, char similarity against box
ground truth:

| Engine | Size | Avg sim | Avg ms |
|---|---|---:|---:|
| arbo | tiny | 93.3% | ~165 |
| arbo | **small** | **94.4%** | ~530 |
| arbo | medium | 94.8% | ~2120 |
| ppu-paddle-ocr (ref) | small | 94.6% | ~793 |

Read the `small` and `medium` rows together: **medium buys ~+0.4 pts for
roughly 4x the CPU latency.** That is why `small` is now the default for both
the library and the CLI. Medium is still the right call when you have GPU or
TensorRT headroom and have measured a real win on *your* documents — but it is
not the first thing to try when accuracy looks wrong.

The same logic applies one level up: `detLimitSideLen` is not "bigger is
better" either. 960 beat 1536 here. If your pages are posters or A3 scans
rather than receipts, re-measure before assuming our number transfers.

## If you are upgrading

The two changes most likely to surprise an existing integration are the
reading order (`page.lines` is now top-to-bottom, left-to-right, so anything
matching boxes by index against an old dump will break) and
`minimumConfidence = 0.5` (weak boxes now disappear from `page.lines`; set it
to `0` for audit dumps that need every box).

The full upgrade checklist, including the settings table and the seven things
to watch when integrating, is in
[Accuracy defaults](../../models/accuracy-defaults.md).
