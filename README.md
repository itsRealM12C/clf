# CLF — Color Format Extractor

A single-file, dependency-free browser tool for decoding headerless raw video frame dumps into viewable/exportable images. Built for reverse-engineering pipelines where a frame buffer has been pulled straight from memory, a capture device, or a firmware dump with no container, no header, and no metadata — just a flat pixel array.

---

## Problem Statement

Raw frame buffers extracted from hardware (camera ISPs, display controllers, embedded firmware, memory dumps) are typically dumped as bare pixel arrays. There is no file signature, no header, and no explicit width/height/format tag — the only ground truth is the byte count and, at best, a naming convention left behind by whoever dumped it. Standard image tooling (`file`, PIL, ImageMagick) cannot open these because there is nothing to sniff.

To recover the image, three unknowns must be resolved:

1. **Pixel format** — how bytes map to color components
2. **Resolution** — how the linear byte stream maps to a 2D grid
3. **Component order / colorspace transform** — byte order within a pixel, and (for YUV) the correct chroma reconstruction

CLF solves (1) via brute-force size matching against a fixed resolution, and (3) via known-correct transform math validated against reference samples. Resolution is currently pinned rather than inferred (see [Constraints](#constraints)).

---

## Detection Strategy

Given a fixed target resolution of **1920×1080** (2,073,600 pixels), each candidate format has a deterministic expected byte count:

| Format | Bytes/pixel | Expected size | Formula |
|---|---|---|---|
| RGBA32 | 4 | 8,294,400 | `W × H × 4` |
| RGB24  | 3 | 6,220,800 | `W × H × 3` |
| YUYV 4:2:2 | 2 (avg) | 4,147,200 | `W × H × 2` |

Because these three byte-per-pixel ratios (4, 3, 2) produce non-overlapping file sizes at a fixed resolution, exact byte-length matching is a sufficient discriminator — no header inspection, entropy analysis, or content heuristics are required. The detector short-circuits on an exact match:

```js
function detectFormat(byteLength) {
  if (byteLength === W*H*4) return 'rgba32';
  if (byteLength === W*H*3) return 'rgb24';
  if (byteLength === W*H*2) return 'yuyv';
  return null;
}
```

If a file's size doesn't match any candidate exactly, detection fails closed — CLF reports the mismatch rather than guessing or padding/truncating data, since a silent partial decode would produce a plausible-looking but geometrically wrong image (shear/skew artifacts from row misalignment).

---

## Format Decoders

### RGB24 → RGBA8888 (canvas-native)
Straight 3→4 channel expansion, alpha forced opaque:
```
[R, G, B] → [R, G, B, 255]
```

### RGBA32 → RGBA8888
Identity passthrough — no transform needed, direct `TypedArray.set()` for a bulk memcpy rather than a per-pixel loop.

### YUYV 4:2:2 → RGBA8888
YUYV (also called YUY2) packs **two luma samples per one chroma pair**, i.e. 4 bytes encode 2 horizontally adjacent pixels:

```
byte:   [ Y0 ][ U  ][ Y1 ][ V  ]
pixel:     P0            P1
```

Both `P0` and `P1` share the same `(U, V)` chroma pair (horizontal chroma subsampling — half the horizontal color resolution, full vertical and full luma). Each pixel is reconstructed independently using its own Y with the shared U/V via the BT.601 YCbCr→RGB matrix:

```
R = Y + 1.402   × (V - 128)
G = Y - 0.344136 × (U - 128) - 0.714136 × (V - 128)
B = Y + 1.772   × (U - 128)
```

Output is clamped to `[0, 255]` per channel. This is the standard SDTV/BT.601 coefficient set (as opposed to BT.709 for HD or BT.2020 for UHD) — chosen because it matched the reference frames used during development; see [Colorspace Caveat](#colorspace-caveat) below.

**Byte-order note:** YUYV and UYVY are frequently confused since both are 4:2:2 packed formats with identical byte *counts* but different *ordering*:
- `YUYV`: `Y0 U Y1 V` (used here)
- `UYVY`: `U Y0 V Y1`

CLF assumes YUYV. If your source pipeline emits UYVY, swap the extraction indices in the decode loop (`data[j]` ↔ `data[j+1]`, `data[j+2]` ↔ `data[j+3]`).

---

## Colorspace Caveat

The BT.601 coefficients above are hardcoded. If your source is HD/UHD video captured under BT.709 or BT.2020, colors will be visibly off (typically slightly desaturated reds/blues). There is no colorspace auto-detection here — raw buffers carry no colorimetry metadata, so this is inherent to the problem, not a bug. If you know your capture pipeline's colorspace differs, the coefficients in the YUYV branch of `decode()` are the only thing that needs changing.

---

## Architecture

```
┌─────────────┐     ┌──────────────┐     ┌───────────────┐     ┌────────────┐
│  drag/drop 　   │ --> │  FileReader  │ --> 　 │  detectFormat  │ --> │   decode   │
│  or <input> 　  │     │ (ArrayBuffer)│    　  │ (size compare) │     │ (per-pixel │
└─────────────┘     └──────────────┘     └───────────────┘     │  transform)│
                                                                　 　  └─────┬──────┘
                                                                    　    ▼
                                                          ┌────────────────────┐
                                                          │  Canvas ImageData  　 │
                                                          │  (RGBA8888 buffer) 　 │
                                                          └─────────┬──────────┘
                                                                    ▼
                                                          ┌────────────────────┐
                                                          │  canvas.toBlob()　 　  │
                                                          │  → object URL   　    │
                                                          │  → <a download>  　   │
                                                          └────────────────────┘
```

Everything runs client-side. No network requests, no server, no external dependencies — it's a single `.html` file that works offline from `file://` or any static host.

### State model
Multiple files can be loaded simultaneously; each is held in memory as a raw `ArrayBuffer` inside `state.files[]`. Switching the active file re-runs detection + decode against the cached buffer rather than re-reading from disk.

### Why `canvas.toBlob()` + object URL instead of `toDataURL()`
`toDataURL()` produces a base64 data URI, which works for direct `<a href>` downloads in a normal top-level browsing context. In sandboxed iframe contexts (e.g. embedded tool previews), triggering a download via a data URI on a detached anchor element can silently fail depending on the sandbox's `allow-downloads` policy and event-trust model. The blob + `URL.createObjectURL()` + DOM-attached-anchor + programmatic `.click()` + `URL.revokeObjectURL()` cleanup pattern is the more portable approach across embedded/sandboxed rendering contexts.

---

## Constraints

- **Resolution is pinned to 1920×1080.** This is a deliberate simplification, not a limitation of the underlying math — the decoders themselves are resolution-agnostic. Supporting arbitrary resolutions would reintroduce the ambiguity problem this tool intentionally avoids (a given byte count is consistent with many `W×H` factor pairs, several formats, and possibly padding/stride). If your dumps come from a different resolution, change the `W`/`H` constants at the top of the `<script>` block.
- **No stride/alignment handling.** Some capture hardware pads each row to a 16- or 32-byte boundary. This tool assumes tightly packed rows (`stride == width × bytes_per_pixel`) with no padding.
- **No multi-frame / video support.** One buffer in, one frame out. If your dump is a concatenated sequence of frames, split it externally first (fixed-size chunking is trivial given a known per-frame byte count).
- **YUYV only, not UYVY/NV12/I420/etc.** Deliberately scoped down per current requirements — the format-selection logic and decode dispatch (`decode(buf, fmt)`) already generalize cleanly if more formats are added back later.

---

## Usage

1. Visit it by [clicking here](https://itsrealm12c.github.io/clf) in any modern browser — no build step, no install.
2. Drag one or more raw dump files onto the drop zone (or click to browse).
3. CLF checks each file's byte length against the three known-good sizes and decodes automatically.
4. If a file doesn't match any expected size, an inline error reports the expected vs. actual byte count rather than failing silently.
5. Click **Download PNG** to export the currently previewed frame as a lossless PNG via the browser's native canvas encoder.

---

## Extending

To add a new fixed-size format:

1. Add a byte-count branch to `detectFormat()`.
2. Add a matching decode branch to `decode()` that writes into the shared `Uint8ClampedArray` RGBA output buffer.
3. Add a human-readable label to `FORMAT_LABELS`.

The rest of the pipeline (file handling, canvas rendering, PNG export) is format-agnostic and requires no changes.
