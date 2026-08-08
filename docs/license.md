---
title: License
---

# License

arboOCR is licensed under the
[Apache License 2.0](https://github.com/wafik/ArboOCR/blob/main/LICENSE).

The ported detection/recognition logic derives from
[RapidOcrOnnx](https://github.com/RapidAI/RapidOcrOnnx) (Apache-2.0) and bundles
[Clipper](https://www.angusj.com/clipper2/) (Boost Software License 1.0). See
[`THIRD_PARTY_NOTICES.md`](https://github.com/wafik/ArboOCR/blob/main/THIRD_PARTY_NOTICES.md)
for full attribution.

## Components

| Component | Upstream | License |
|---|---|---|
| arboOCR | [wafik/ArboOCR](https://github.com/wafik/ArboOCR) | [Apache License 2.0](https://github.com/wafik/ArboOCR/blob/main/LICENSE) |
| Ported detection / recognition logic | [RapidOcrOnnx](https://github.com/RapidAI/RapidOcrOnnx) | Apache-2.0 |
| Clipper (vendored) | [Clipper2](https://www.angusj.com/clipper2/) | Boost Software License 1.0 |

!!! note "Attribution is not optional"

    Apache-2.0 and BSL-1.0 are both permissive — you can ship arboOCR in a
    closed-source product — but Apache-2.0 requires you to carry the notices
    forward. Redistribute
    [`THIRD_PARTY_NOTICES.md`](https://github.com/wafik/ArboOCR/blob/main/THIRD_PARTY_NOTICES.md)
    with your binaries; it is the authoritative list, kept current with what
    the repository actually vendors.

## The model weights

The Components table covers the library source. arboOCR still does not bundle
the PP-OCRv6 model files — they are fetched separately and are not part of this
repository — but it now ships a **default download URL** pointing at weights the
project hosts itself, at
[ARBO-TEAM/arbo-ocr-models](https://github.com/ARBO-TEAM/arbo-ocr-models). Since
those are the bytes you get if you do nothing, arboOCR is the distributor of
them, and owes you the provenance rather than deferring to someone else's terms.

The weights derive from
[PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR)'s PP-OCR models, and
upstream PaddleOCR is Apache-2.0. The models repository carries a `NOTICE`
recording that provenance and licence; treat it as authoritative for the
weights, the same way `THIRD_PARTY_NOTICES.md` is authoritative for the code.

!!! info "Two licences, two scopes"

    arboOCR's Apache-2.0 covers arboOCR's own code. It makes no claim about the
    weights, and the weights' terms make no claim about the code — so if you
    redistribute a product containing both, carry both notices. If you point
    `modelsDir` at ONNX files you obtained yourself, arboOCR downloads nothing
    and the models repository's `NOTICE` never enters the picture.

See [Models](models/index.md).
