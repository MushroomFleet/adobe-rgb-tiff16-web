# True Adobe RGB TIFF Export — Implementation Checklist

Reusable checklist for adding **color-managed 16-bit Adobe RGB (1998) TIFF export** to any image pipeline. Derived from the working `export-tiff-web` reference implementation.

**Status**: Applied to `ComfyUI-Save-TIFF` — see `color_space.py`, `tiff_encoder.py`, `nodes.py`

---

## Prerequisites

- [x] Source images are in **sRGB** (standard for web, ComfyUI, most GPU renderers)
- [x] Pipeline can access raw pixel data as float or integer arrays
- [x] Output format is **TIFF 6.0** (uncompressed RGB)

---

## 1. Color Space Conversion (sRGB → Adobe RGB)

> Implemented in `color_space.py` — `convert_srgb_to_adobe_rgb()` (numpy vectorized)

The pixel values must be mathematically transformed — you cannot just tag sRGB data as Adobe RGB.

- [x] **Normalize** pixel values to `[0.0, 1.0]` float range
- [x] **Linearize sRGB** — undo the sRGB transfer function (piecewise: linear segment below 0.04045, gamma 2.4 above)
  ```
  if v <= 0.04045:  linear = v / 12.92
  else:             linear = ((v + 0.055) / 1.055) ^ 2.4
  ```
- [x] **Matrix multiply** — convert linear sRGB to linear Adobe RGB via combined `XYZ_TO_ADOBE_RGB * SRGB_TO_XYZ` matrix:
  ```
  | 0.7151627  0.2848373  0.0000000 |   | R_lin |
  | 0.0000000  1.0000000  0.0000000 | × | G_lin |
  | 0.0000000  0.0413880  0.9586120 |   | B_lin |
  ```
  Both spaces use D65 white point — no chromatic adaptation needed.
- [x] **Clamp** linear Adobe RGB values to `[0.0, 1.0]`
- [x] **Apply Adobe RGB gamma** — encode with gamma `563/256 = 2.19921875`:
  ```
  output = v ^ (256/563)    (i.e., the inverse gamma)
  ```
- [x] **Quantize** to target bit depth (16-bit: `round(v * 65535)`, 8-bit: `round(v * 255)`)

---

## 2. Build Adobe RGB (1998) ICC Profile

> Implemented in `color_space.py` — `build_adobe_rgb_icc_profile()`, cached via `get_adobe_rgb_icc_profile()`

A valid ICC v2.1 profile must be embedded so readers interpret colors correctly.

- [x] **Header (128 bytes)**
  - Profile size, preferred CMM `ADBE`, version `2.1.0`
  - Device class `mntr`, color space `RGB `, PCS `XYZ `
  - Creation date: `1999-06-03`
  - Platform `APPL`, profile signature `acsp`
  - PCS illuminant D50: `(0.9642, 1.0000, 0.8249)` as s15Fixed16
  - Profile creator `ADBE`

- [x] **Tag table (9 tags)**
  | Tag    | Purpose                                 |
  |--------|-----------------------------------------|
  | `desc` | Profile description: `"Adobe RGB (1998)"` |
  | `wtpt` | White point XYZ: `(0.9505, 1.0, 1.0890)` — D65 |
  | `rXYZ` | Red primary (D50-adapted): `(0.6097559, 0.3111145, 0.0194702)` |
  | `gXYZ` | Green primary (D50-adapted): `(0.2052401, 0.6256714, 0.0608902)` |
  | `bXYZ` | Blue primary (D50-adapted): `(0.1492240, 0.0632141, 0.7445396)` |
  | `rTRC` | Tone reproduction curve: gamma `2.19921875` (u8Fixed8 = `0x0233`) |
  | `gTRC` | Same curve (shared offset with `rTRC`) |
  | `bTRC` | Same curve (shared offset with `rTRC`) |
  | `cprt` | Copyright text |

- [x] All XYZ values stored as **s15Fixed16** (big-endian, `round(val * 65536)`)
- [x] All multi-byte ICC fields are **big-endian** (unlike TIFF which is little-endian)
- [x] Tag data offsets 4-byte aligned
- [x] Profile can be cached/reused across exports (it's static)

---

## 3. TIFF Encoder — Embed ICC Profile

> Implemented in `tiff_encoder.py` — `encode_tiff()`

The TIFF file must include the ICC profile via the standard tag.

- [x] **TIFF tag 34675** (`ICC_PROFILE`) — type `BYTE`, count = profile byte length, value = offset to profile data
- [x] Profile bytes written into the TIFF file body at the declared offset
- [x] IFD entry count incremented to account for the ICC tag (e.g., 11 → 12 tags)
- [x] IFD entries remain **sorted by tag number** (TIFF 6.0 requirement)
- [x] Pixel data offset recalculated to account for ICC profile size
- [x] If writing 16-bit data, ensure pixel data offset is **2-byte aligned** after ICC block

---

## 4. TIFF Structure Reference

Minimum required IFD tags for a valid uncompressed RGB TIFF:

| Tag   | Name              | Value                          |
|-------|-------------------|--------------------------------|
| 256   | ImageWidth        | pixel width                    |
| 257   | ImageLength       | pixel height                   |
| 258   | BitsPerSample     | `(16, 16, 16)` or `(8, 8, 8)` |
| 259   | Compression       | `1` (none)                     |
| 262   | PhotometricInterp | `2` (RGB)                      |
| 273   | StripOffsets      | offset to pixel data           |
| 277   | SamplesPerPixel   | `3`                            |
| 278   | RowsPerStrip      | image height (single strip)    |
| 279   | StripByteCounts   | `W * H * 3 * bytesPerSample`  |
| 282   | XResolution       | `72/1` (rational)              |
| 283   | YResolution       | `72/1` (rational)              |
| 34675 | ICCProfile        | offset to ICC data             |

- [x] Byte order: little-endian (`II`, `0x4949`)
- [x] Magic number: `42`
- [x] Pixel data: interleaved RGB, no alpha channel

---

## 5. Validation

- [ ] Open exported TIFF in **Photoshop** — verify "Adobe RGB (1998)" appears in color profile info
- [ ] Open in **GIMP** — confirm ICC profile is detected on import
- [ ] Run `exiftool` or `tiffinfo` — verify ICC Profile tag is present and profile description reads `"Adobe RGB (1998)"`
- [ ] Compare colors visually: Adobe RGB export should appear slightly desaturated in an unmanaged viewer (expected — the gamut is wider, values are lower for same visual color)
- [ ] Round-trip test: export Adobe RGB TIFF, re-import, convert back to sRGB — colors should match original within quantization error

---

## 6. Common Pitfalls

- **Tagging without converting**: Labeling sRGB data as Adobe RGB shifts all colors. Always convert pixel values first.
- **Wrong gamma**: Adobe RGB uses `2.19921875` (not `2.2`). The exact value is `563/256`.
- **Endianness mismatch**: ICC profiles are always big-endian. TIFF data is per-header (typically little-endian). Don't mix them.
- **Missing profile**: Without the ICC tag, viewers assume sRGB and colors render incorrectly.
- **Alignment**: 16-bit pixel data must start on a 2-byte boundary. Pad after ICC data if needed.
- **`imageio.imwrite()` alone is not enough**: Libraries like `imageio` or `Pillow` may write valid TIFF but won't embed a custom ICC profile unless explicitly told to.

---

## Quick Reference: Pipeline Flow

```
Source pixels (sRGB uint8/float)
  │
  ├─ sRGB path ──────── strip alpha ──► quantize ──► TIFF (no ICC tag)
  │
  └─ Adobe RGB path ──► normalize [0,1]
                           │
                           ▼
                        linearize (undo sRGB gamma)
                           │
                           ▼
                        3×3 matrix (sRGB linear → Adobe RGB linear)
                           │
                           ▼
                        clamp [0,1]
                           │
                           ▼
                        apply Adobe RGB gamma (v^(256/563))
                           │
                           ▼
                        quantize (×65535 for 16-bit)
                           │
                           ▼
                        TIFF encoder + embed ICC profile (tag 34675)
```

---

## Implementation Map

| File | Module | What it does |
|------|--------|-------------|
| `color_space.py` | `convert_srgb_to_adobe_rgb()` | Sections 1 — full sRGB→Adobe RGB pixel conversion (numpy vectorized) |
| `color_space.py` | `build_adobe_rgb_icc_profile()` | Section 2 — builds ICC v2.1 profile from scratch (big-endian, 9 tags) |
| `color_space.py` | `get_adobe_rgb_icc_profile()` | Section 2 — cached profile accessor |
| `tiff_encoder.py` | `encode_tiff()` | Sections 3–4 — TIFF 6.0 binary encoder with ICC tag 34675 |
| `nodes.py` | `SaveTiff.save_images()` | Pipeline glue — color space dropdown, conversion, encode, write |
