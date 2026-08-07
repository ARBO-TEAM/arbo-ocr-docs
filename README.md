# arbo-ocr-docs

[![Check Links](https://github.com/ARBO-TEAM/arbo-ocr-docs/actions/workflows/links-check.yml/badge.svg)](https://github.com/ARBO-TEAM/arbo-ocr-docs/actions/workflows/links-check.yml)

Source for the **arboOCR** documentation site, built with
[MkDocs](https://www.mkdocs.org/) and
[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/).

- **Published site:** <https://arbo-team.github.io/arbo-ocr-docs/>
- **Library repo:** <https://github.com/wafik/ArboOCR>

This repo contains documentation only. Bug reports and feature requests for
the OCR library itself belong in the library repo.

## Local development

```bash
pip install -r requirements.txt
mkdocs serve
```

Then open <http://127.0.0.1:8000>. The dev server live-reloads on save.

The `git-committers` and `git-revision-date-localized` plugins are disabled
unless `CI` is set, so `mkdocs serve` stays fast and does not burn an
unauthenticated GitHub API rate limit.

## Publishing

Deployment is handled by GitHub Actions and versioned with
[mike](https://github.com/jimporter/mike):

- **Push to `main`** — rebuilds and deploys the `latest` alias.
- **Push a `v*` tag** — publishes a versioned copy of the docs at that tag,
  selectable from the version picker on the site.

Do not commit to `gh-pages` by hand; mike owns that branch.

## Repo layout

Pages live under `docs/` and mirror the site nav — top-level guides at the
root (`index.md`, `quickstart.md`, `faq.md`) and one directory per section
(`build/`, `models/`, `api/`, `wrappers/`), plus `docs/blog/` for posts and
`docs/stylesheets/extra.css` for theme overrides. `mkdocs.yml` holds the nav,
theme, plugin and markdown-extension configuration — a new page is not
visible until it is added to the `nav` block there. CI lives in
`.github/workflows/`: the develop and release build workflows plus a
scheduled link checker (`links-check.yml`, with exclusions in
`.lycheeignore`).
