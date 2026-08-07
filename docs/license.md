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

The table above covers the library source. arboOCR does not bundle the PP-OCRv6
model files — they are fetched separately and are not part of this repository,
so their terms are whatever their own distributor sets. See
[Models](models/index.md).
