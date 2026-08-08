---
date:
  created: 2026-08-08
authors:
  - arbo-team
categories:
  - Release notes
tags:
  - models
  - api
description: >-
  arboOCR downloads its own weights now. That reverses a decision we reaffirmed
  yesterday — so here is what actually changed, and what did not.
---

# We said there would never be a default model URL

Yesterday's post has a section titled
[The documented download path produced a broken models directory](2026-08-07-library-improvements.md).
It fixed the downloader's missing dictionary file, and on the way through it
restated the standing position: **arboOCR ships no default URL; the caller
supplies `baseUrl`.** That sentence is now false.

`ensureOcrModels(cfg)` fetches missing weights from a default location,
verifies them against a checksum compiled into the binary, and caches them in a
platform-correct directory. First run downloads. Every run after that is
offline.

This is a reversal, one day old. It is worth being precise about which part of
the old position gave way, because it was not the reasoning.

<!-- more -->

## The old position was three objections wearing one coat

"There is no default URL and there never will be" was never really one
argument. Unpacked, it was three, and each of them was a genuine failure mode:

| Objection | The failure it names |
|---|---|
| **Rot** | A URL hardcoded into a released binary outlives whoever maintains the other end of it. Branches move, buckets get repointed, and a v1.4 binary starts 404ing two years after anyone can rebuild it. |
| **Integrity** | "The download succeeded" and "the bytes are correct" are different claims. A truncated `.onnx` mmap'd into ONNX Runtime is not a clean error; it is a crash in somebody else's stack frame. |
| **Provenance** | Pointing users at weights makes you responsible for what those weights are and what licence they carry. Not knowing is not a defence. |

Shipping a default without answers to all three would have been worse than
shipping none — a convenience that quietly transfers three risks to the caller
and does not mention it. Declining was the right call *at the time it was
made*.

What changed is that all three now have answers, and the answers turned out to
be cheap. None of them required a new dependency. Below is each one, and what
it cost.

## Rot: an immutable tag is a version, not a moving target

The default is:

```text
https://github.com/ARBO-TEAM/arbo-ocr-models/releases/download/models-v1/
```

The load-bearing part is `models-v1`. It is a release tag, not a branch. Assets
attached to a tag do not change under a binary that was built expecting them,
which converts the URL from a moving target into something closer to a version
pin. A future `models-v2` is a new tag and a new default in a new release — it
does not reach backwards and alter what an already-shipped binary resolves.

The weights live in a new repo,
[ARBO-TEAM/arbo-ocr-models](https://github.com/ARBO-TEAM/arbo-ocr-models), as
**release assets rather than Git LFS**. That was a deliberate rejection, not a
default:

| | Git LFS (free tier) | Release assets |
|---|---|---|
| Bandwidth | 1 GB/month | No cap |
| Per-file limit | 2 GB | 2 GB |
| Pulls of the full ~178 MB model set | ~5/month before throttling | Unmetered |

Five pulls a month is not a distribution channel. It is a tripwire, and the
first CI matrix that fans out across three OSes trips it before lunch. Release
assets have no bandwidth cap, and the 2 GB per-file ceiling is nowhere near
binding for a model set that totals 178 MB.

## Integrity: a compiled-in hash, and a rename that cannot tear

A SHA-256 for **every stock file** is compiled into the arboOCR binary. That
turns the check from "did the transfer finish" into "are these the exact bytes
we tested against", which is the only version of the question worth asking.

`downloadFile` never writes to the destination path. It writes to a sibling
`.<pid>.tmp`, hashes the completed temporary file, and only on a match calls
`std::filesystem::rename` into place — atomic on POSIX and on Win32 alike. Two
consequences fall out of that, both of which were the point:

- A reader **never observes a partial file.** The model path either does not
  exist or is a complete, verified model. There is no window in which it is
  half a tensor.
- Two processes racing the same model **cannot corrupt each other.** The `.tmp`
  names differ by PID, so both write independently and both rename; the loser
  of the race replaces a byte-identical file.

The part that matters more in practice: an **already-present file is re-hashed
rather than trusted.** A cache is not evidence. If a previous run died mid-write
under an older build, or a disk filled, or a container layer got truncated, the
file on disk is wrong and every subsequent run inherits it. Verified directly —
a cached recognizer truncated to 24 bytes was refetched to its full
**4,489,813 bytes** with a matching hash, rather than being handed to ONNX
Runtime to fail on.

!!! tip "This is the failure mode that has no good error message"
    A corrupt `.onnx` does not surface as "corrupt `.onnx`". It surfaces as an
    ONNX Runtime protobuf parse error, or a segfault inside a memory-mapped
    read, several frames away from anything you control. Re-hashing on every
    run costs a few milliseconds and removes the entire class.

## Seventy lines of SHA-256, not a crypto dependency

The obvious way to get SHA-256 is OpenSSL. The obvious way is wrong here.

arboOCR needs exactly one hash function, in one file, for one purpose. Linking
a full TLS and crypto stack for that means a new vcpkg port, a new set of
platform build failures, and a new CVE feed to track — all to compute a digest
that fits in seventy lines.

So it is vendored: roughly **70 lines in `model_downloader.cpp`**, tested
against the NIST vectors, including the 56-byte input that lands exactly on the
padding boundary — the one case a hand-written implementation gets wrong. Test
vectors are the reason writing this yourself is defensible; without them it
would not be.

`vcpkg.json` is unchanged by this entire feature. That was a constraint going
in, not a happy accident.

## The cache directory is platform-correct and tag-scoped

Two independent things had to be right here.

**Platform-correct**, because `~/.cache` is an XDG convention and Windows is not
an XDG platform. A tool that creates `C:\Users\<user>\.cache\` on Windows is
leaving litter outside the directory the OS designates for exactly this, and it
will survive every profile-migration and cleanup tool that knows about
`%LOCALAPPDATA%`.

**Tag-scoped**, because the cache key has to include the model set version.
Otherwise a `models-v2` release that reuses a filename silently loads a
`models-v1` file, and the resulting accuracy regression is invisible — the file
exists, it parses, it just is not the model you asked for.

| Platform | Path |
|---|---|
| Windows | `%LOCALAPPDATA%\arboOCR\models\models-v1` |
| macOS | `~/Library/Caches/arboOCR/models/models-v1` |
| Linux | `$XDG_CACHE_HOME/arboOCR/models/models-v1`, else `~/.cache/arboOCR/models/models-v1` |

## Your explicit path is still never substituted

The escape hatch that used to be the *only* option is unchanged, and it still
wins. `ensureOcrModels(cfg)` resolves **per file**, in this order:

1. An explicitly set `detModelPath` / `recModelPath` / etc. is returned **as-is**
   and is never substituted by a download. If you point at a fine-tuned
   recognizer, you get your recognizer — it has no stock checksum and is not
   expected to have one.
2. An existing file under `modelsDir` wins. A populated models directory means
   **zero network**, which is what makes this a no-op for every existing
   deployment.
3. Only then does it download.

`EngineConfig` gained two fields — `bool autoDownload = true` and
`std::string modelsBaseUrl` — and nothing else moved.

```cpp
arboOCR::EngineConfig cfg;

// Air-gapped: never touch the network, fail loudly if a file is missing.
cfg.autoDownload = false;

// Or point at an internal mirror. Stock filenames stay checksum-verified,
// so a mirror serving different bytes is rejected, not accepted quietly.
cfg.modelsBaseUrl = "https://models.internal.example/arboocr/models-v1/";
```

The same controls exist outside C++, because for the Go, Rust, and PHP wrappers
the CLI *is* the API:

| Control | Effect |
|---|---|
| `--no-download` / `ARBOOCR_OFFLINE=1` | Never fetch. Missing model is an error, not a download. |
| `--models-url` / `ARBOOCR_MODELS_URL` | Internal mirror. Stock names remain checksum-verified. |
| `--download-models` | Prefetch every stock file and exit. |

`--download-models` exists for one specific reason:

```dockerfile
# Weights land in the image layer, so the container never needs egress.
RUN arboocr_demo --download-models
```

Same trick in a CI cache step. The alternative — every job in the matrix
pulling 178 MB on first inference — is the thing that makes people distrust
auto-download in the first place.

!!! warning "A mirror does not turn off verification"
    `--models-url` changes where stock files are fetched from. It does not
    change what they must hash to. If your mirror is stale or has been tampered
    with, the fetch fails rather than silently loading different weights. If you
    genuinely need different weights, that is what an explicit path is for.

## What we learned from a peer project

The push to revisit this came from reading ppu-paddle-ocr — the TypeScript
PaddleOCR wrapper that already appears in our accuracy tables as a reference
point — which self-hosts its models and downloads them on demand.

It got the hard part right, and the hard part was the one blocking us: **it
solved availability by hosting the weights itself** rather than pointing at
somebody else's URL and hoping. That is the move. Once you accept that you have
to host the bytes, the "hardcoded URL rots" objection stops being about trust in
a third party and becomes a versioning problem you control — which is a much
smaller problem, and the one an immutable tag solves.

Reading its downloader closely also surfaced four things arboOCR's
implementation was then shaped to avoid. None of these are bugs in the sense of
"does not work" — they work fine on the happy path. They are the edges:

| Edge | Consequence | What arboOCR does |
|---|---|---|
| No checksum anywhere in the source | A truncated or substituted download is indistinguishable from a good one | SHA-256 per stock file, compiled in, re-checked on cache hits |
| URLs pinned to a mutable `main` branch | The bytes a released version resolves can change under it | Immutable release tag, `models-v1` |
| Cache keyed on `basename(url)` alone | Two different URLs sharing a filename collide in one cache slot | Cache path scoped by tag, not by filename alone |
| Plain `writeFileSync`, no temp-and-rename | An interrupted write leaves a partial file that looks complete | Write to `.<pid>.tmp`, hash, then atomic `rename` |

Its cache also lands in `C:\Users\<user>\.cache\` on Windows rather than
`%LOCALAPPDATA%` — the same XDG-on-Windows assumption described above.

The useful framing is not that one implementation is careless and the other
careful. It is that a downloader has a long tail of states that never show up in
manual testing — half-written files, two processes, a stale mirror, a filename
collision — and every one of them has to be designed for deliberately, because
none of them will ever fail during development.

## Hosting the weights makes us a distributor

This is the obligation the old posture sidestepped. "We ship no URL, so the
weights are not our problem" is a real answer, and it stops being available the
moment you host the bytes.

So the weights had to be identified rather than assumed. They are **paddle2onnx
conversions of PaddlePaddle's released PP-OCRv6 inference models**, and that is
established from the graphs themselves, not from where they were found:

```text
# Node names carry the paddle2onnx prefix.
p2o.pd_op.conv2d.0
p2o.pd_op.batch_norm.3

# Initializers keep Paddle's original tensor naming.
conv2d_0.w_0
batch_norm2d_1.b_0
```

An ONNX file that had been through a different conversion path, or that had been
retrained, would not carry both. Upstream PaddleOCR is Apache-2.0, and the models
repo carries a `NOTICE` recording the conversion and the upstream licence.

This is the part of the reversal with an ongoing cost attached. The URL and the
hashes are code — write them once and they keep working. Being a distributor is
a standing commitment, and it is the one that has to be re-honoured every time
the model set changes.

## The preconditions changed, not the reasoning

The three objections in the first section are still correct. A hardcoded URL
still rots, an unverified download is still not an integrity guarantee, and
hosting weights still carries a licensing obligation. Nothing in this post
argues otherwise.

What changed is that each one now has a specific, cheap answer:

- Rot → an immutable release tag, so the URL is a version.
- Integrity → a compiled-in hash manifest, so "resolved" becomes "correct".
- Provenance → a `NOTICE` recording what the weights are and where they came
  from.

Had we shipped a default URL last week, we would have shipped it without those,
and we would have been wrong. Shipping it today is not a change of mind about
what a default URL owes you. It is that we finally paid it.

Full behaviour, including the precedence rules and every opt-out, is on
[Model downloader](../../api/downloader.md); the flags are on
[Command line](../../cli.md).
