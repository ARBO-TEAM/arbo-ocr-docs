---
title: Markdown export
---

# Markdown export

One function, for the moment when a flat list of lines is not what you wanted —
you wanted the document.

```cpp
#include <arboOCR/markdown.hpp>

std::string toMarkdown(const PagePrediction& page);
```

OCR gives you lines, not structure. Where `page.lines` breaks is an artifact of
how wide the page was, not a fact about the text: one sentence wrapped over
three lines arrives as three entries, a heading arrives as a line no different
in shape from body text, and a receipt's `TOTAL` and `12.50` arrive either fused
into one string with a gap in the middle or as two separate boxes sitting on the
same row. `toMarkdown` reads the structure back out of the geometry that is
already in the result — line heights, vertical gaps, left edges, vertical
overlap, and the spacing between words — and emits markdown.

It never throws, and an empty page yields an empty string. It expects
`page.lines` in reading order, which
[`Engine::recognize`](index.md#arboocrengine) already guarantees (centroid y,
then x).

## Three surfaces

=== "C++"

    ```cpp
    #include <arboOCR/engine.hpp>
    #include <arboOCR/markdown.hpp>
    #include <fstream>

    arbo::ocr::EngineConfig cfg;
    cfg.modelsDir = "models";
    cfg.returnWordBoxes = true;   // only needed for narrow-gutter `key | value` rows

    arbo::ocr::Engine engine(cfg);
    auto page = engine.recognize("invoice.png");

    std::ofstream out("invoice.md", std::ios::binary);
    out << arbo::ocr::toMarkdown(page);
    ```

=== "CLI"

    ```bash
    arboocr_demo --image invoice.png --models-dir models --markdown invoice.md
    ```

    ```text
    Backend: cpu
    Markdown: invoice.md
    Image: invoice.png
    Lines: 11 (339.338 ms)
    ```

    `--markdown <path>` composes with `--json` exactly the way `--draw` does —
    the file is written first, its confirmation and any failure go to stderr, so
    stdout stays a single parseable object:

    ```bash
    arboocr_demo --image invoice.png --models-dir models \
      --json --markdown invoice.md > invoice.json
    ```

    A write failure is a warning, not an error: recognition itself succeeded, so
    the exit code and the JSON on stdout are unaffected.

=== "Python"

    ```python
    from arboocr import Engine, EngineConfig, to_markdown

    cfg = EngineConfig()
    cfg.models_dir = "models"
    cfg.return_word_boxes = True

    page = Engine(cfg).recognize("invoice.png")

    with open("invoice.md", "w", encoding="utf-8") as f:
        f.write(to_markdown(page))
    ```

    `to_markdown(page)` mirrors `to_json(page)`: one call, one string, no
    options.

## What it infers

| Feature | How it is detected | What it emits |
|---|---|---|
| **Paragraph** | Consecutive text lines with a small vertical gap *and* a shared left edge belong to one block. | The block's lines joined into a single paragraph with one space per seam — not a hard break — separated from the next block by a blank line. |
| **Heading** | A block of **at most two** text lines whose median height clears a multiple of the page's median line height. Two tiers. | `# Heading` for the tallest tier, `## Heading` for the next. A block that is already a list or a key/value row is never promoted, however large its type. |
| **List item** | The text begins with `-`, `*`, `•` (U+2022) or `·` (U+00B7), or with a digit run or single letter followed by `.` or `)`. The marker must be followed by a space or end the line. | `- item` / `1. item`, one per line — list items are **not** merged into the surrounding paragraph. Every bullet glyph collapses to `-`, and `2)` is re-spelled `2.`, the form markdown counts from. |
| **Key/value row** *(word-level)* | **One** detected line whose `words` show a single gap far wider than the line's own typical inter-word gap. Needs word boxes. | A `\| key \| value \|` table row. |
| **Key/value row** *(line-level)* | **Two or more consecutive lines** that overlap vertically by at least half a line height and are separated by a wide horizontal gutter. Needs no word boxes at all. | The same `\| key \| value \|` row, from the same splitter and the same emitter. |
| **Table** | A run of two or more adjacent key/value rows, from either detector. | One GFM table. A run of exactly one row does **not** become a table — see below. |
| **Block break** | A vertical gap above the paragraph threshold, or a change of left edge. Between two key/value rows the vertical bar is raised, because form rows are set with far more air than prose. | A blank line, starting a new block. |

The marker rule is deliberately strict: `-5.00` on a receipt is a negative
amount, not a bullet, and `1.5kg` is not item one.

Lines whose `text` is empty are dropped before anything is measured, so a blank
recognition cannot invent a vertical gap between the two real lines it sits
between. A line with no usable polygon has no geometry to compare and stays with
whatever it followed, rather than inventing a break by pretending it sits at the
origin.

### Two ways a key/value row is found

The same visual row reaches `toMarkdown` in one of two shapes, depending on how
wide the gutter was, and there is a detector for each. Both feed the same
splitter, produce the same internal item, and flow through the same block
grouping and the same emitter — so there is one set of thresholds, not two.

| | Word-level | Line-level |
|---|---|---|
| **Page shape** | Narrow gutter | Wide gutter |
| **What the detector did** | Emitted **one** box containing label and value | Emitted **two separate** boxes on the same row |
| **Where the gap shows up** | Between that line's `words` | Between two consecutive `LinePrediction`s |
| **Needs `returnWordBoxes`** | Yes | No |

!!! tip "The line-level path is the common one"
    A real column gutter is wide enough that DBNet splits the label and the
    value into separate boxes, which means the gap never appears between one
    line's words and the word-level pass cannot see it at all. Most invoices and
    forms are recovered by the line-level detector — including the one in
    [Before and after](#before-and-after) below.

    This is also why the line-level pass has to run *before* block grouping: the
    left-edge rule would otherwise file a label at `x=40` and its value at
    `x=520` as two different blocks, which is exactly how a wide-gutter invoice
    used to come out as alternating one-line paragraphs.

### A lone pair is not a table

A run of **one** key/value row is emitted as `**label** value`, not as a table.

A one-row GFM table is a header plus a separator with no body, which renders as
an empty table — strictly worse than the text it replaced. A single wide gap is
also weak evidence on its own. Bolding the label keeps both halves and keeps the
split visible without claiming a column structure exists.

Two or more adjacent rows do become a table, and the **first pair becomes the
header row**. GFM has no table without a header; synthesizing an empty one would
render as a blank band above the data, while promoting the first row invents no
text that was not on the page — and on invoices that row genuinely is the column
caption more often than not.

### Two columns out, always

A row with three or more pieces is **not** expanded into three or more cells. It
splits at its single widest gap: everything to the left of that gap is the
label, everything to the right is the value.

```text
1  Cheeseburger      12.00
```

```markdown
| 1 Cheeseburger | 12.00 |
```

Nothing is dropped — the cap never strands a piece. Agreeing on a column count
across the rows of a run is real cell-grid reconstruction, which is
[out of scope](#what-it-does-not-do).

The comparison that picks the split point is made against the median of the
row's *other* gaps, not of all of them. This is what keeps fully justified prose
out of the table path — every inter-word space there is stretched, so all the
gaps grow together and none of them dwarfs the rest — and what makes the line
item above split after the description rather than after the quantity.

!!! note "Joining pieces in CJK"
    When the label or value is assembled from several pieces, a space is
    inserted only when both sides of the seam are ASCII. CJK recognition gives
    every character its own word box, and joining those with spaces would shred
    the text; this gets both scripts right without carrying a Unicode word-break
    table. Paragraph joining, by contrast, always uses a single space.

## Thresholds are fractions of the median line height

Every geometric threshold in `markdown.cpp` is expressed as a multiple of the
page's **median line height**. There is no pixel constant in the file.

| Constant | Value | What it gates |
|---|---|---|
| `kBlockGapFraction` | `0.8` | Vertical gap that ends a paragraph block. |
| `kRowGapFraction` | `2.0` | Vertical gap that ends a run of key/value rows. |
| `kIndentFraction` | `1.5` | Left-edge shift that starts a new block. |
| `kRowOverlapFraction` | `0.5` | Vertical overlap two boxes need to count as one visual row. |
| `kHeading1Ratio` | `1.6` | Block height ÷ page median at or above which a block becomes `#`. |
| `kHeading2Ratio` | `1.25` | The same, for `##`. |
| `kKeyValueGapFraction` | `1.5` | Absolute bar a gap must clear to be a gutter rather than a space. |
| `kKeyValueGapRatio` | `3.0` | How far the candidate gap must dwarf the row's other gaps. |

The one constant that is not a fraction is `kMaxHeadingLines = 2`, a line count:
past two lines a tall block is a large-print paragraph, and promoting it would
swallow the body text.

!!! tip "This is a resolution guarantee, not a coincidence"
    Because the unit is the page's own median line height, the same page
    photographed at 96 DPI and scanned at 300 DPI produces **byte-identical**
    markdown, and no knob has to be re-tuned per source. The values above are
    chosen against the physical facts they measure — a space glyph is a quarter
    to a third of an em, so `1.5` line heights of blank is four to six spaces
    wide and cannot be word spacing; single-spaced prose leaves `0.2`–`0.5` line
    heights between glyph boxes while a real paragraph break starts at ~`1.2`,
    so `0.8` sits in the empty middle. This was learned the hard way:
    `sortLinesReadingOrder` once used a hard-coded 12 px row tolerance, which
    fragmented 300 DPI scans and merged thumbnails.

## Before and after

A rendered invoice: a heading in large bold, two body lines set close together,
three label/value rows with a wide gutter, and two bullets.

```bash
arboocr_demo --image invoice.png --models-dir models --markdown out.md
```

The engine hands back eleven lines. The columns that matter are each line's left
edge and which lines share a visual row — the wide gutter made the detector emit
the label and the value as two separate boxes:

```text
#   text                                                      x0    row
1   INVOICE                                                   40     a      (taller than the page median)
2   This document confirms the purchase of goods listed       40     b
3   below and is valid without signature.                     40     c
4   Invoice No                                                40     d
5   INV-2026-0042                                            520     d
6   Date                                                      40     e
7   07 August 2026                                           520     e
8   Total                                                     40     f
9   149000                                                   520     f
10  • Delivery within 3 days                                  40     g
11  • Payment on receipt                                      40     h
```

`out.md`:

```markdown
## INVOICE

This document confirms the purchase of goods listed below and is valid without signature.

| Invoice No | INV-2026-0042 |
| --- | --- |
| Date | 07 August 2026 |
| Total | 149000 |

- Delivery within 3 days
- Payment on receipt
```

Four things to read out of that:

- **Lines 2 and 3 became one paragraph.** They were one sentence in the source
  document, arrived as two OCR lines, and left as one. That merge is the whole
  point — a flat line list preserves the page's wrapping, which is exactly the
  information you do not want when the text is headed for a diff, an embedding,
  or a language model.
- **Line 1 became `##`, not `#`.** Its height cleared the `1.25` tier but not
  the `1.6` one. The tier is the type size on the page, not the line's position
  in the document.
- **Rows `d`, `e` and `f` were paired by the line-level detector**, and the
  first pair — `Invoice No` / `INV-2026-0042` — became the table's header row.
  There is no separate caption line on the page; GFM simply has no table without
  a header, so the first row is promoted rather than a blank one being invented.
- **The bullet glyphs normalized to `-`.** Whatever the recognizer read — `•`,
  `·`, `*` or `-` — comes out as the one bullet markdown spells.

## What it escapes, and what it deliberately does not

The escaping policy is narrow on purpose: touch only what would silently change
the document, and leave everything else as the recognizer read it.

| Input | Escaped? | Why |
|---|---|---|
| `\|` | **Inside table cells only** | There it ends the cell. Everywhere else it is an ordinary character and stays one. |
| `]` | **Only when `](` or `][` follows** | That is the one shape that turns OCR text into a link and eats the target. A bare `[1]` is a shortcut reference with no definition, so it renders literally — citations and footnote markers stay readable instead of becoming `\[1\]`. |
| CR, LF, TAB | **Always**, replaced with a space | The only inline characters that can shatter block structure from inside a line. |
| Leading `#` or `>` | **Always** | Would open a heading or a blockquote nobody asked for. |
| Leading `-`, `*` or `+` followed by a space or end of line | **Always** | Would open a bullet list. |
| A line consisting **entirely** of `-`, `=`, `_` or `*` (plus spaces) | **Always** | A receipt's `--------` separator is a setext heading underline. Left alone it silently retitles the paragraph printed above it — this is the case that actually bites. |
| `*`, `_`, `#` and backticks **mid-line** | **Never** | Unpaired they are literal; paired they cost at worst italics or a code span, with the text still on the page. Escaping them would turn the masked card number `**** 1234` that is on half the receipts arboOCR targets into `\*\*\*\* 1234`. |

The split is between *structure-forming positions* — the start of a line, and
the inside of a table cell — and everything else. Only the former can change
what the document *is*; the latter can at worst change how a few characters
look, and unreadable escapes are a worse trade than a stray italic.

## `--markdown` implies word boxes

The word-level key/value detector measures the gap *between words*, so it needs
[`LinePrediction::words`](index.md#word-boxes). The CLI therefore turns
`returnWordBoxes` on whenever `--markdown` is passed, whether or not you also
passed `--word-boxes`.

!!! note "This does not change your `--json` payload"
    `--json --markdown out.md` still emits the JSON shape you would get without
    `--markdown`. The CLI turns word boxes on internally for the markdown pass
    and then drops them again before serializing, because `"words"` appearing
    *iff* the caller asked for it is a published contract — `--markdown` must
    not silently reshape a wrapper's output. Pass `--word-boxes` explicitly if
    you want them in the JSON.

In C++ and Python nothing is implied for you — set `returnWordBoxes` /
`return_word_boxes` yourself before constructing the `Engine`. Without it,
`toMarkdown` still works, and it loses less than you might expect: headings,
paragraphs and lists are all line-level, and so is the **line-level** key/value
detector, which is the one that recovers wide-gutter invoices and forms. What
you lose is only the narrow-gutter case — a label and value the detector fused
into one box. Those lines stay plain paragraph lines, with the internal gap
collapsed to a single space.

!!! warning "Word polygons are approximate"
    Word boxes come from **CTC timestep alignment**, a by-product of
    recognition rather than a supervised output. They run roughly **half a
    character wide of true glyph extents**, consistently lagging right — which
    is why the gap measurement clamps negative gaps to zero, since neighbouring
    boxes can overlap by a hair. That is fine for the one thing `toMarkdown`
    asks of them — deciding *where* a gap is and whether it is unusually wide —
    but it is not pixel-exact, and the same caveat applies to anything else you
    build on `words`. See the [full discussion](index.md#word-boxes).

## What it does not do

!!! warning "Rough by design, and rough in specific known places"
    - **Multi-column page bodies come out mangled. This is a known, accepted
      limitation, not an oversight.** A two-column article is geometrically
      *identical* to a page of key/value rows: a ragged-right left column leaves
      gaps every bit as wide as an invoice gutter, so neither the absolute
      gutter width nor its share of the row width discriminates between them.
      No per-row test exists. A page-level "nearly every line pairs up" guard
      would catch articles, but it would equally break a full-page form — which
      *is* the shape arboOCR targets — so the miss is deliberately left on the
      article side. The damage is also done upstream regardless: `page.lines` is
      sorted by centroid y then x, so a two-column layout arrives already
      interleaved across the columns before `toMarkdown` ever sees it.
      Newspapers, two-column papers, and side-by-side forms come out wrong.
    - **No table-grid reconstruction.** Only the two-column `key | value` shape
      is recovered, and rows with more pieces are capped at two columns rather
      than expanded. Ruled tables, spanned cells, and genuine N-column grids are
      not reconstructed.
    - **No image, figure, or horizontal-rule detection.** Non-text content
      never produced a line in the first place, so it is simply absent from the
      output. There is no placeholder and no warning.
    - **Geometric heuristics degrade quietly.** Skewed, curved, warped, or
      rotated text does not fail loudly — it yields plausible-looking markdown
      with the wrong structure, which is harder to notice than an error.

    This is the same envelope RapidOCR draws around its own `to_markdown`, whose
    documentation calls it 粗略 — crude. Treat the result as a better starting
    point than a flat list of lines, not as a faithful document conversion. If
    you need real layout analysis, you need a layout model, and arboOCR does not
    ship one.

## See also

- [API Reference](index.md) — `Engine`, `PagePrediction`, and the JSON output.
- [Word boxes](index.md#word-boxes) — what `words` is, and why its geometry is approximate.
- [CLI reference](../cli.md) — `--markdown`, `--json`, `--word-boxes`, and how they compose.
- [Visualization](visualize.md) — the other export, for when you need to see where the boxes landed rather than what the text says.
