# Adobe RGB TIFF16 Web — Browser-Based 16-Bit TIFF Image Exporter

A fully client-side, browser-based 16-bit TIFF export tool with Adobe RGB (1998) color space support. No server, no backend, no installation — runs entirely in your browser.

**Live App:** [https://scuffedepoch.com/adobe-tiff16/](https://scuffedepoch.com/adobe-tiff16/)

## What It Does

Drop any image (JPEG, PNG, WebP, BMP) into the browser and export it as an uncompressed 16-bit TIFF file with correct color management. Choose between **sRGB** passthrough or **Adobe RGB (1998)** output with mathematically correct gamut conversion through CIE XYZ D65 and an embedded ICC profile.

This tool back-engineers the `SaveTiff` node from [ComfyUI-HQ-Image-Save](https://github.com/spacepxl/ComfyUI-HQ-Image-Save) into a pure browser implementation. The original Python node converts floating-point image tensors (0.0–1.0) to 16-bit unsigned integers (0–65535) and writes uncompressed TIFF files via `imageio`. This web adaptation replicates that pipeline entirely in JavaScript/TypeScript — the TIFF binary is constructed from scratch using the TIFF 6.0 specification, producing files byte-compatible with what `imageio.imwrite()` would output for the same pixel data.

### Key Features

- Drag-and-drop or file-picker image loading (JPEG, PNG, WebP, BMP)
- Persistent image storage via IndexedDB (survives page reloads)
- Gallery view of all stored images with thumbnails
- 16-bit TIFF export replicating the ComfyUI SaveTiff pipeline exactly
- **Color space selection: sRGB or Adobe RGB (1998)** with correct gamut conversion and embedded ICC profile
- Configurable filename prefix (mirrors the original `filename_prefix` parameter)
- Batch export of multiple images with auto-incrementing filenames
- Optional 8-bit export mode for smaller file sizes
- Zero server dependency — runs entirely in the browser

### Adobe RGB (1998) Pipeline

When Adobe RGB is selected, the export performs:

1. **Linearize sRGB** — undo the sRGB transfer function (piecewise: linear segment below 0.04045, power curve 2.4 above)
2. **Linear sRGB to CIE XYZ D65** — 3x3 matrix multiplication
3. **CIE XYZ D65 to Linear Adobe RGB** — 3x3 matrix multiplication (pre-composed into a single combined matrix)
4. **Clamp** linear values to [0.0, 1.0]
5. **Apply Adobe RGB gamma** — raise each channel to 1/2.19921875 (= 256/563)
6. **Embed ICC profile** — standard Adobe RGB (1998) ICC v2.1 profile via TIFF tag 34675 (InterColorProfile)

The resulting TIFF is correctly interpreted by Photoshop, Lightroom, GIMP, and any ICC-aware application.

## Implementation Plan

The full TINS implementation plan used to build this application is included in this repository:

- [`export-tiff-web-plan.md`](./export-tiff-web-plan.md) — Complete specification covering image loading, IndexedDB storage, TIFF 6.0 encoding, color space conversion, ICC profile construction, and UI component architecture.

## Technology

Built with Vite + React + TypeScript. No third-party TIFF library — the encoder is entirely self-contained. The color space conversion and ICC profile builder are written from scratch against the ICC specification.

## License

MIT

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{adobe_rgb_tiff16_web,
  title = {Adobe RGB TIFF16 Web: Browser-Based 16-Bit TIFF Image Exporter with Adobe RGB Color Space Support},
  author = {Drift Johnson},
  year = {2025},
  url = {https://github.com/MushroomFleet/adobe-rgb-tiff16-web},
  version = {1.0.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
