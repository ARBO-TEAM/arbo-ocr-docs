---
title: Contributing
---

# Contributing

Issues and PRs welcome on [wafik/ArboOCR](https://github.com/wafik/ArboOCR).

## Citing upstream when you port or compare

If you're porting from or comparing against the upstream RapidOcrOnnx/PaddleOCR
algorithms, please cite the specific upstream source/commit you're comparing
against — this project tracks those correspondences closely (see doc comments
and
[`THIRD_PARTY_NOTICES.md`](https://github.com/wafik/ArboOCR/blob/main/THIRD_PARTY_NOTICES.md)).

!!! tip "Why a commit, not just a repo name"

    "This matches RapidOcrOnnx" ages badly — upstream moves, and a year later
    nobody can tell which behaviour was matched or whether a later upstream fix
    applies here too. A file path plus a commit SHA makes that answerable. The
    existing doc comments in `src/arboOCR/` are the model to follow.

Before opening a PR, build and run the test suite for your target — see
[Build](build/index.md) and, on aarch64, [Jetson / aarch64](build/jetson.md).

## Contributing to these docs

The documentation site lives in its own repository:
[ARBO-TEAM/arbo-ocr-docs](https://github.com/ARBO-TEAM/arbo-ocr-docs).

```bash
pip install -r requirements.txt
mkdocs serve
```

- Pages are Markdown files under `docs/`.
- Navigation is defined in the `nav:` section of `mkdocs.yml` — a new page is
  not reachable until it is listed there.
- Every page on this site has an edit pencil in the top right that opens the
  correct source file on GitHub, which is the fastest route for a typo or a
  one-paragraph fix.
