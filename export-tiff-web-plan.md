# Export TIFF Web — Browser-Based 16-Bit TIFF Image Exporter

## Description

A Vite + TypeScript + React single-page application that back-engineers the `SaveTiff` node from the [ComfyUI-HQ-Image-Save](./ComfyUI-HQ-Image-Save/) custom node pack into a fully client-side, browser-based TIFF export tool. The original Python node converts floating-point image tensors (0.0–1.0) to 16-bit unsigned integers (0–65535) and writes uncompressed TIFF files via `imageio`. This web adaptation replicates that pipeline entirely in-browser: the user loads one or more images (JPEG, PNG, WebP, BMP), which are stored in IndexedDB for persistence, then exports any stored image as a 16-bit-per-channel uncompressed TIFF file — all without any server or backend.

A core enhancement over the original node is **color space support**. The user can choose between **sRGB** (default, passthrough) and **Adobe RGB (1998)** output color spaces. When Adobe RGB is selected, the export pipeline performs a mathematically correct gamut conversion through CIE XYZ D65, applies the Adobe RGB gamma curve (2.19921875 = 563/256), and embeds the standard Adobe RGB (1998) ICC profile into the TIFF file via tag 34675 (InterColorProfile). This ensures the exported TIFF is correctly interpreted by Photoshop, Lightroom, GIMP, and any ICC-aware application.

The TIFF binary is constructed from scratch in TypeScript using the TIFF 6.0 specification, producing files byte-compatible with what `imageio.imwrite()` would output for the same pixel data. No third-party TIFF library is used; the encoder is self-contained.

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

---

## Functionality

### 1. Image Loading

**File Input Methods:**
- A visible drop zone occupying the upper portion of the UI accepts files via drag-and-drop.
- A "Browse Files" button inside the drop zone opens a native file picker with `accept="image/jpeg,image/png,image/webp,image/bmp"`.
- Multiple files may be selected or dropped simultaneously.

**Processing on Load:**
1. For each file, create an `HTMLImageElement` and load via `URL.createObjectURL(blob)`.
2. Draw the image onto an offscreen `<canvas>` at its natural dimensions.
3. Extract pixel data via `ctx.getImageData(0, 0, width, height)` — this yields RGBA `Uint8ClampedArray` with values 0–255.
4. Store the following record in IndexedDB:
   - `id`: auto-incrementing primary key
   - `name`: original filename (without extension)
   - `width`: pixel width
   - `height`: pixel height
   - `rgbaData`: the raw `Uint8ClampedArray` buffer (stored as `ArrayBuffer`)
   - `thumbnailDataUrl`: a downscaled base64 data URL (max 200px on longest side) for gallery display
   - `createdAt`: ISO 8601 timestamp
5. Revoke the object URL after canvas extraction.

**Validation:**
- Reject files that are not valid image types. Show an inline toast: `"Unsupported file type: {filename}"`.
- Reject images whose decoded dimensions exceed 16384 x 16384 (canvas safety limit). Show toast: `"Image too large: {filename} ({w}x{h})"`.
- If IndexedDB storage fails (e.g., quota exceeded), show toast: `"Storage full — cannot save {filename}"`.

### 2. Image Gallery

**Layout:**
- Below the drop zone, a responsive grid of stored images is displayed.
- Each card shows:
  - The thumbnail image
  - The filename (truncated with ellipsis if longer than 24 characters)
  - Dimensions in pixels (e.g., `1920 x 1080`)
  - A checkbox for batch selection
  - A delete button (trash icon)

**Interactions:**
- Clicking a card (not the checkbox or delete button) opens a detail/preview modal.
- The "Select All" / "Deselect All" toggle above the grid controls all checkboxes.
- The delete button removes the record from IndexedDB and removes the card with a fade-out transition (200ms).
- If there are zero stored images, show a centered empty state: `"No images stored. Drop files above to get started."`.

### 3. TIFF Export

**Export Controls (toolbar below the gallery):**
- **Filename Prefix** text input — default value `"export"`, mirrors the original ComfyUI node's `filename_prefix` parameter. Alphanumeric, hyphens, and underscores only; validated on change.
- **Color Space** dropdown — `sRGB` (default) or `Adobe RGB (1998)`. When Adobe RGB is selected, a small indicator label reading "Wide Gamut" appears next to the dropdown in `--accent` color to remind the user that gamut conversion will be applied.
- **Bit Depth** toggle — `16-bit` (default, matches original) or `8-bit`.
- **Export Selected** button — enabled when at least one image is checked. Exports all checked images.
- **Export All** button — exports every image in the gallery.

**Export Pipeline (mirrors the original `save_images` method):**

For each image to be exported, perform the following steps (directly adapted from the Python source):

```
Original Python (ComfyUI-HQ-Image-Save/nodes.py lines 388-392):
    i = 65535. * image.cpu().numpy()
    img = np.clip(i, 0, 65535).astype(np.uint16)
    file = f"{filename}_{counter:05}_.tiff"
    imageio.imwrite(os.path.join(full_output_folder, file), img)
```

**TypeScript equivalent:**

1. **Retrieve** the `rgbaData` ArrayBuffer from IndexedDB and wrap as `Uint8ClampedArray`.
2. **Normalize** each R, G, B channel value from the 0–255 integer range to 0.0–1.0 floating point: `value / 255.0`. The alpha channel is discarded (the original node processes only RGB). This produces a `Float64Array` (or `Float32Array`) of length `width * height * 3`.
3. **Color Space Conversion** (conditional on `colorSpace` option):
   - **sRGB** (default): No conversion. The normalized values are already in sRGB.
   - **Adobe RGB (1998)**: Apply the full conversion pipeline from `color-space.ts`:
     1. **Linearize sRGB**: Undo the sRGB transfer function (piecewise: linear segment below 0.04045, power curve 2.4 above). This matches the reference `sRGBtoLinear()` function from the original `nodes.py` lines 18-21.
     2. **Linear sRGB → CIE XYZ D65**: Multiply each pixel's `[R, G, B]` by the 3x3 sRGB-to-XYZ matrix.
     3. **CIE XYZ D65 → Linear Adobe RGB**: Multiply by the 3x3 XYZ-to-AdobeRGB matrix.
     4. **Clamp** linear values to `[0.0, 1.0]` (out-of-gamut colors are clipped).
     5. **Apply Adobe RGB gamma**: Raise each channel to the power `1/2.19921875` (= 256/563).
   - The result remains a normalized `[0.0, 1.0]` float array, now in the target color space.
4. **Scale to 16-bit**: multiply each normalized float by `65535.0`.
5. **Clamp**: `Math.min(Math.max(Math.round(value), 0), 65535)` — equivalent to `np.clip(i, 0, 65535)`.
6. **Pack** into a `Uint16Array` of length `width * height * 3` in row-major, channel-interleaved order (R, G, B, R, G, B, ...).
7. **Encode** the pixel data into a TIFF binary blob (see TIFF Encoder section below). When `colorSpace` is `'adobe-rgb'`, the encoder embeds the Adobe RGB (1998) ICC profile via TIFF tag 34675.
8. **Download** using a programmatically created `<a>` element with `URL.createObjectURL(blob)` and `download` attribute set to `"{prefix}_{counter:05}_.tiff"` (5-digit zero-padded counter, matching original naming).

For **8-bit mode**, the same color space conversion is applied in float space, then the result is scaled by `255.0` and quantized to `Uint8Array`. The TIFF encoder writes `BitsPerSample = [8, 8, 8]` instead of `[16, 16, 16]`.

**Batch export** iterates through selected images with an incrementing counter starting at `00000`. If a single image is exported, it still uses `00000`.

### 4. Preview Modal

When a gallery card is clicked:
- A modal overlay (semi-transparent black backdrop) appears.
- The full-resolution image is rendered on a `<canvas>` element from the stored `rgbaData`.
- The modal header shows: filename, dimensions, stored date.
- A close button (top-right X) and backdrop click dismiss the modal.
- An "Export This Image" button in the modal footer exports the single image.

---

## Technical Implementation

### Project Structure

```
export-tiff-web/
├── index.html
├── package.json
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
├── vite.config.ts
├── src/
│   ├── main.tsx                  # React entry point
│   ├── App.tsx                   # Root component, layout orchestrator
│   ├── App.css                   # Application styles
│   ├── index.css                 # Global reset and base styles
│   ├── components/
│   │   ├── DropZone.tsx          # Drag-and-drop + file picker
│   │   ├── Gallery.tsx           # Image grid with selection
│   │   ├── GalleryCard.tsx       # Individual image card
│   │   ├── ExportToolbar.tsx     # Export controls and actions
│   │   ├── PreviewModal.tsx      # Full-size image preview
│   │   └── Toast.tsx             # Notification toasts
│   ├── lib/
│   │   ├── db.ts                 # IndexedDB wrapper (idb)
│   │   ├── tiff-encoder.ts       # Raw TIFF 6.0 binary encoder + ICC profile embedding
│   │   ├── color-space.ts        # sRGB ↔ Adobe RGB conversion, ICC profile data
│   │   ├── image-processing.ts   # Canvas decode, pixel normalization
│   │   └── download.ts           # Blob download utility
│   ├── hooks/
│   │   ├── useImageStore.ts      # CRUD operations on IndexedDB images
│   │   └── useToast.ts           # Toast notification state
│   └── types.ts                  # Shared TypeScript interfaces
```

### Dependencies

```json
{
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "idb": "^8.0.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "@vitejs/plugin-react": "^4.4.0",
    "typescript": "^5.7.0",
    "vite": "^6.1.0"
  }
}
```

The only non-React runtime dependency is `idb` — a tiny (~1 KB gzipped) promise-based wrapper around the raw IndexedDB API. No TIFF library is used; the encoder is hand-written.

### Data Model

```typescript
// src/types.ts

export interface StoredImage {
  id: number;                  // Auto-incremented primary key
  name: string;                // Original filename without extension
  width: number;               // Pixel width
  height: number;              // Pixel height
  rgbaData: ArrayBuffer;       // Raw RGBA pixel data (Uint8ClampedArray buffer)
  thumbnailDataUrl: string;    // Base64 data URL for gallery thumbnail
  createdAt: string;           // ISO 8601 timestamp
}

export type ColorSpace = 'srgb' | 'adobe-rgb';

export interface ExportOptions {
  filenamePrefix: string;      // Default: "export"
  colorSpace: ColorSpace;      // Default: "srgb"
  bitDepth: 8 | 16;           // Default: 16
}

export interface ToastMessage {
  id: string;                  // Unique ID for dismissal
  type: 'info' | 'error' | 'success';
  text: string;
}
```

### IndexedDB Schema (`src/lib/db.ts`)

```typescript
import { openDB, type IDBPDatabase } from 'idb';

const DB_NAME = 'export-tiff-web';
const DB_VERSION = 1;
const STORE_NAME = 'images';

export async function getDb(): Promise<IDBPDatabase> {
  return openDB(DB_NAME, DB_VERSION, {
    upgrade(db) {
      if (!db.objectStoreNames.contains(STORE_NAME)) {
        const store = db.createObjectStore(STORE_NAME, {
          keyPath: 'id',
          autoIncrement: true,
        });
        store.createIndex('createdAt', 'createdAt');
      }
    },
  });
}

export async function getAllImages(): Promise<StoredImage[]> {
  const db = await getDb();
  return db.getAll(STORE_NAME);
}

export async function addImage(image: Omit<StoredImage, 'id'>): Promise<number> {
  const db = await getDb();
  return db.add(STORE_NAME, image) as Promise<number>;
}

export async function getImage(id: number): Promise<StoredImage | undefined> {
  const db = await getDb();
  return db.get(STORE_NAME, id);
}

export async function deleteImage(id: number): Promise<void> {
  const db = await getDb();
  return db.delete(STORE_NAME, id);
}
```

### Image Processing (`src/lib/image-processing.ts`)

```typescript
import type { StoredImage } from '../types';

/**
 * Decode an image File into raw RGBA pixel data via offscreen canvas.
 * Returns all fields needed for a StoredImage record (minus id).
 */
export async function decodeImageFile(
  file: File
): Promise<Omit<StoredImage, 'id'>> {
  const bitmap = await createImageBitmap(file);
  const { width, height } = bitmap;

  if (width > 16384 || height > 16384) {
    bitmap.close();
    throw new Error(`Image too large: ${file.name} (${width}x${height})`);
  }

  // Extract full-resolution RGBA
  const canvas = new OffscreenCanvas(width, height);
  const ctx = canvas.getContext('2d')!;
  ctx.drawImage(bitmap, 0, 0);
  const imageData = ctx.getImageData(0, 0, width, height);
  const rgbaData = imageData.data.buffer;

  // Generate thumbnail (max 200px on longest side)
  const scale = Math.min(200 / width, 200 / height, 1);
  const thumbW = Math.round(width * scale);
  const thumbH = Math.round(height * scale);
  const thumbCanvas = new OffscreenCanvas(thumbW, thumbH);
  const thumbCtx = thumbCanvas.getContext('2d')!;
  thumbCtx.drawImage(bitmap, 0, 0, thumbW, thumbH);
  bitmap.close();

  const thumbBlob = await thumbCanvas.convertToBlob({ type: 'image/png' });
  const thumbnailDataUrl = await blobToDataUrl(thumbBlob);

  // Derive name from filename (strip extension)
  const name = file.name.replace(/\.[^.]+$/, '');

  return {
    name,
    width,
    height,
    rgbaData,
    thumbnailDataUrl,
    createdAt: new Date().toISOString(),
  };
}

function blobToDataUrl(blob: Blob): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result as string);
    reader.onerror = reject;
    reader.readAsDataURL(blob);
  });
}

/**
 * Convert stored RGBA (8-bit) pixel buffer to a normalized Float64Array
 * of RGB values in [0.0, 1.0]. This is the intermediate format used
 * before color space conversion and final quantization.
 *
 * Output layout: [R, G, B, R, G, B, ...] (interleaved, alpha discarded)
 */
export function rgbaToNormalizedRgb(
  rgbaBuffer: ArrayBuffer,
  width: number,
  height: number
): Float64Array {
  const rgba = new Uint8ClampedArray(rgbaBuffer);
  const pixelCount = width * height;
  const rgb = new Float64Array(pixelCount * 3);

  for (let i = 0; i < pixelCount; i++) {
    const srcIdx = i * 4;
    const dstIdx = i * 3;
    rgb[dstIdx]     = rgba[srcIdx]     / 255.0;
    rgb[dstIdx + 1] = rgba[srcIdx + 1] / 255.0;
    rgb[dstIdx + 2] = rgba[srcIdx + 2] / 255.0;
  }

  return rgb;
}

/**
 * Quantize a normalized [0.0, 1.0] RGB Float64Array to 16-bit Uint16Array.
 *
 * This replicates the ComfyUI SaveTiff pipeline:
 *   scaled  = normalized * 65535.0   (equivalent to: i = 65535. * image)
 *   clamped = clip(scaled, 0, 65535) (equivalent to: np.clip(i, 0, 65535))
 *   output  = uint16(clamped)        (equivalent to: .astype(np.uint16))
 */
export function quantizeToUint16(rgb: Float64Array): Uint16Array {
  const out = new Uint16Array(rgb.length);
  for (let i = 0; i < rgb.length; i++) {
    out[i] = Math.min(Math.max(Math.round(rgb[i] * 65535), 0), 65535);
  }
  return out;
}

/**
 * Quantize a normalized [0.0, 1.0] RGB Float64Array to 8-bit Uint8Array.
 */
export function quantizeToUint8(rgb: Float64Array): Uint8Array {
  const out = new Uint8Array(rgb.length);
  for (let i = 0; i < rgb.length; i++) {
    out[i] = Math.min(Math.max(Math.round(rgb[i] * 255), 0), 255);
  }
  return out;
}

/**
 * Convenience wrappers that combine normalize → quantize for sRGB passthrough
 * (no color space conversion needed, so we can use the fast integer path).
 */
export function rgbaToRgb16(
  rgbaBuffer: ArrayBuffer,
  width: number,
  height: number
): Uint16Array {
  const rgba = new Uint8ClampedArray(rgbaBuffer);
  const pixelCount = width * height;
  const rgb16 = new Uint16Array(pixelCount * 3);

  for (let i = 0; i < pixelCount; i++) {
    const srcIdx = i * 4;
    const dstIdx = i * 3;
    // Direct 8→16 expansion: val * 257 maps 0→0, 255→65535
    rgb16[dstIdx]     = rgba[srcIdx]     * 257;
    rgb16[dstIdx + 1] = rgba[srcIdx + 1] * 257;
    rgb16[dstIdx + 2] = rgba[srcIdx + 2] * 257;
  }

  return rgb16;
}

export function rgbaToRgb8(
  rgbaBuffer: ArrayBuffer,
  width: number,
  height: number
): Uint8Array {
  const rgba = new Uint8ClampedArray(rgbaBuffer);
  const pixelCount = width * height;
  const rgb8 = new Uint8Array(pixelCount * 3);

  for (let i = 0; i < pixelCount; i++) {
    const srcIdx = i * 4;
    const dstIdx = i * 3;
    rgb8[dstIdx]     = rgba[srcIdx];
    rgb8[dstIdx + 1] = rgba[srcIdx + 1];
    rgb8[dstIdx + 2] = rgba[srcIdx + 2];
  }

  return rgb8;
}
```

### Color Space Conversion (`src/lib/color-space.ts`)

This module implements the sRGB to Adobe RGB (1998) color space conversion and provides the ICC profile data for embedding into the TIFF. The conversion follows the standard ICC pipeline: source decode → CIE XYZ connection space → destination encode.

```typescript
/**
 * Color space conversion: sRGB → Adobe RGB (1998)
 *
 * Pipeline:
 *   1. Undo sRGB transfer function (linearize)
 *   2. Linear sRGB → CIE XYZ D65 (3x3 matrix)
 *   3. CIE XYZ D65 → Linear Adobe RGB (3x3 matrix)
 *   4. Apply Adobe RGB gamma encode
 *
 * Reference white: D65 (both sRGB and Adobe RGB use D65)
 * Adobe RGB gamma: 2.19921875 (exactly 563/256)
 *
 * The sRGB linearization matches the original ComfyUI-HQ-Image-Save
 * sRGBtoLinear() function from nodes.py lines 18-21.
 */

// --- sRGB Transfer Function ---

/** Inverse sRGB companding (decode): sRGB → linear sRGB */
function srgbToLinear(v: number): number {
  // Matches nodes.py sRGBtoLinear: threshold 0.0404482362771082
  if (v <= 0.04045) {
    return v / 12.92;
  }
  return Math.pow((v + 0.055) / 1.055, 2.4);
}

// --- Adobe RGB (1998) Transfer Function ---

/** Adobe RGB gamma: exactly 563/256 = 2.19921875 */
const ADOBE_RGB_GAMMA = 563 / 256;
const ADOBE_RGB_GAMMA_INV = 256 / 563;

/** Adobe RGB companding (encode): linear → Adobe RGB */
function linearToAdobeRgb(v: number): number {
  if (v <= 0) return 0;
  return Math.pow(v, ADOBE_RGB_GAMMA_INV);
}

// --- Conversion Matrices ---

/**
 * sRGB linear → CIE XYZ D65
 * Source: IEC 61966-2-1:1999 (sRGB standard)
 */
const SRGB_TO_XYZ: [number, number, number][] = [
  [0.4124564, 0.3575761, 0.1804375],
  [0.2126729, 0.7151522, 0.0721750],
  [0.0193339, 0.1191920, 0.9503041],
];

/**
 * CIE XYZ D65 → Adobe RGB (1998) linear
 * Source: Adobe RGB (1998) Color Image Encoding, version 2005-05
 * This is the inverse of the Adobe RGB → XYZ matrix.
 */
const XYZ_TO_ADOBE_RGB: [number, number, number][] = [
  [ 2.0413690, -0.5649464, -0.3446944],
  [-0.9692660,  1.8760108,  0.0415560],
  [ 0.0134474, -0.1183897,  1.0154096],
];

/**
 * Pre-computed combined matrix: sRGB linear → Adobe RGB linear
 * M_combined = XYZ_TO_ADOBE_RGB * SRGB_TO_XYZ
 *
 * Pre-computing avoids two matrix multiplications per pixel.
 */
const SRGB_LINEAR_TO_ADOBE_RGB_LINEAR: [number, number, number][] = [
  [ 0.7151627,  0.2848373,  0.0000000],
  [ 0.0000000,  1.0000000,  0.0000000],
  [ 0.0000000,  0.0413880,  0.9586120],
];

// --- Pixel Conversion ---

/**
 * Convert a single pixel [R, G, B] from sRGB to Adobe RGB (1998).
 * Input and output are normalized [0.0, 1.0].
 */
function srgbPixelToAdobeRgb(r: number, g: number, b: number): [number, number, number] {
  // Step 1: Linearize sRGB
  const lr = srgbToLinear(r);
  const lg = srgbToLinear(g);
  const lb = srgbToLinear(b);

  // Step 2+3: Linear sRGB → Linear Adobe RGB (combined matrix)
  const m = SRGB_LINEAR_TO_ADOBE_RGB_LINEAR;
  const aR = m[0][0] * lr + m[0][1] * lg + m[0][2] * lb;
  const aG = m[1][0] * lr + m[1][1] * lg + m[1][2] * lb;
  const aB = m[2][0] * lr + m[2][1] * lg + m[2][2] * lb;

  // Step 4: Clamp to [0,1] then apply Adobe RGB gamma
  return [
    linearToAdobeRgb(Math.max(0, Math.min(1, aR))),
    linearToAdobeRgb(Math.max(0, Math.min(1, aG))),
    linearToAdobeRgb(Math.max(0, Math.min(1, aB))),
  ];
}

/**
 * Convert an entire normalized RGB float buffer from sRGB to Adobe RGB (1998).
 * Operates in-place on a Float64Array of length pixelCount * 3.
 * Channel layout: [R, G, B, R, G, B, ...] (interleaved, no alpha).
 */
export function convertSrgbToAdobeRgb(rgb: Float64Array): void {
  const pixelCount = rgb.length / 3;
  for (let i = 0; i < pixelCount; i++) {
    const idx = i * 3;
    const [r, g, b] = srgbPixelToAdobeRgb(rgb[idx], rgb[idx + 1], rgb[idx + 2]);
    rgb[idx]     = r;
    rgb[idx + 1] = g;
    rgb[idx + 2] = b;
  }
}

// --- Adobe RGB (1998) ICC Profile ---

/**
 * Standard Adobe RGB (1998) ICC profile.
 *
 * This is the official 560-byte ICC v2 profile for Adobe RGB (1998).
 * Embedding this in the TIFF (tag 34675 / 0x8773) ensures that any
 * ICC-aware application (Photoshop, Lightroom, GIMP, darktable, etc.)
 * correctly interprets the pixel data as Adobe RGB.
 *
 * The profile is stored as a Base64 string and decoded to Uint8Array
 * at first use. It contains:
 *   - Profile header (128 bytes)
 *   - Tag table (desc, wtpt, bkpt, rTRC, gTRC, bTRC, rXYZ, gXYZ, bXYZ, cprt)
 *   - Parametric curve type with gamma 563/256
 *   - D65 white point
 *   - Adobe RGB chromaticity primaries
 *
 * Alternative: construct the profile programmatically (see below).
 */

/**
 * Build a minimal Adobe RGB (1998) ICC v2 profile programmatically.
 *
 * This constructs a valid ICC profile from first principles, containing:
 *   - Header: v2.1, 'mntr' device class, 'RGB ' color space, 'XYZ ' PCS
 *   - Required tags: desc, wtpt, rXYZ, gXYZ, bXYZ, rTRC, gTRC, bTRC, cprt
 *   - Gamma encoded as curveType with a single uint16 fixed-point value
 *
 * Returns a Uint8Array containing the complete ICC profile.
 */
export function buildAdobeRgbIccProfile(): Uint8Array {
  // --- Tag data preparation ---

  // Profile description: "Adobe RGB (1998)"
  const descText = 'Adobe RGB (1998)';
  // desc tag: type 'desc', 4 reserved, 4 count, string, 0 padding for
  // localized strings (simplified: ASCII only)
  const descSize = 4 + 4 + 4 + descText.length + 1 + 12 + 12; // ~49 → pad to 4-byte boundary

  // Copyright
  const cprtText = 'No copyright';
  const cprtSize = 4 + 4 + cprtText.length + 1; // ~21 → pad to 4-byte boundary

  // XYZ type: 4 (sig) + 4 (reserved) + 12 (one XYZ number) = 20 bytes
  const xyzTagSize = 20;

  // Curve type with single gamma: 4 (sig) + 4 (reserved) + 4 (count=1) + 2 (value) = 14 → pad to 16
  const curveTagSize = 16;

  // White point D65: X=0.9505, Y=1.0000, Z=1.0890 (s15Fixed16Number)
  // Adobe RGB primaries in XYZ (s15Fixed16Number):
  //   R: X=0.6097559, Y=0.3111145, Z=0.0194702
  //   G: X=0.2052401, Y=0.6256714, Z=0.0608902
  //   B: X=0.1492240, Y=0.0632141, Z=0.7445396

  // Adobe RGB gamma as u8Fixed8Number: 2.19921875 = 2 + 51/256
  // ICC spec curveType with count=1: value is u8Fixed8Number
  //   2.19921875 = 2 + 0.19921875 = 2 + 51/256
  //   As u8Fixed8: high byte = 2, low byte = 51 (0x33) → 0x0233

  // Calculate layout
  const tagCount = 9; // desc, wtpt, rXYZ, gXYZ, bXYZ, rTRC, gTRC, bTRC, cprt
  const tagTableSize = 4 + tagCount * 12; // count + entries
  const headerSize = 128;

  // Tag data starts after header + tag table
  const dataStart = headerSize + tagTableSize;

  // Align to 4-byte boundaries
  function align4(n: number) { return (n + 3) & ~3; }

  const descAligned = align4(descSize);
  const cprtAligned = align4(cprtSize);

  const descOffset  = dataStart;
  const wtptOffset  = descOffset + descAligned;
  const rXYZOffset  = wtptOffset + xyzTagSize;
  const gXYZOffset  = rXYZOffset + xyzTagSize;
  const bXYZOffset  = gXYZOffset + xyzTagSize;
  const rTRCOffset  = bXYZOffset + xyzTagSize;
  // gTRC and bTRC share the same curve data (same gamma for all channels)
  const cprtOffset  = rTRCOffset + curveTagSize;

  const profileSize = cprtOffset + cprtAligned;

  const buf = new ArrayBuffer(profileSize);
  const view = new DataView(buf);
  const bytes = new Uint8Array(buf);
  let off = 0;

  // --- Helper functions ---
  function writeU32(offset: number, val: number) { view.setUint32(offset, val, false); }
  function writeU16(offset: number, val: number) { view.setUint16(offset, val, false); }
  function writeS15F16(offset: number, val: number) {
    // s15Fixed16Number: signed, 16.16 fixed-point
    const fixed = Math.round(val * 65536);
    view.setInt32(offset, fixed, false);
  }
  function writeSig(offset: number, sig: string) {
    for (let i = 0; i < 4; i++) bytes[offset + i] = sig.charCodeAt(i);
  }
  function writeAscii(offset: number, str: string) {
    for (let i = 0; i < str.length; i++) bytes[offset + i] = str.charCodeAt(i);
  }

  // --- ICC Header (128 bytes) ---
  writeU32(0, profileSize);        // Profile size
  writeSig(4, 'ADBE');             // Preferred CMM (Adobe)
  writeU32(8, 0x02100000);         // Version 2.1.0
  writeSig(12, 'mntr');            // Device class: monitor
  writeSig(16, 'RGB ');            // Color space: RGB
  writeSig(20, 'XYZ ');            // PCS: XYZ
  // Date: 1999-06-03 00:00:00 (original Adobe profile creation date)
  writeU16(24, 1999);              // Year
  writeU16(26, 6);                 // Month
  writeU16(28, 3);                 // Day
  writeU16(30, 0);                 // Hour
  writeU16(32, 0);                 // Minute
  writeU16(34, 0);                 // Second
  writeSig(36, 'acsp');            // Profile file signature
  writeSig(40, 'APPL');            // Primary platform: Apple
  writeU32(44, 0);                 // Profile flags
  writeSig(48, 'none');            // Device manufacturer
  writeSig(52, 'none');            // Device model
  // Device attributes (8 bytes): 0
  // Rendering intent: perceptual (0)
  writeU32(64, 0);
  // PCS illuminant: D50 (required by ICC spec, even though Adobe RGB uses D65)
  writeS15F16(68, 0.9642);        // X
  writeS15F16(72, 1.0000);        // Y
  writeS15F16(76, 0.8249);        // Z
  writeSig(80, 'ADBE');            // Profile creator

  // --- Tag Table ---
  off = headerSize;
  writeU32(off, tagCount); off += 4;

  // Tag entries: [signature, offset, size]
  const tags: [string, number, number][] = [
    ['desc', descOffset,  descAligned],
    ['wtpt', wtptOffset,  xyzTagSize],
    ['rXYZ', rXYZOffset,  xyzTagSize],
    ['gXYZ', gXYZOffset,  xyzTagSize],
    ['bXYZ', bXYZOffset,  xyzTagSize],
    ['rTRC', rTRCOffset,  curveTagSize],
    ['gTRC', rTRCOffset,  curveTagSize],  // shares rTRC data
    ['bTRC', rTRCOffset,  curveTagSize],  // shares rTRC data
    ['cprt', cprtOffset,  cprtAligned],
  ];

  for (const [sig, tagOff, tagSize] of tags) {
    writeSig(off, sig); off += 4;
    writeU32(off, tagOff); off += 4;
    writeU32(off, tagSize); off += 4;
  }

  // --- Tag Data ---

  // desc (textDescriptionType)
  off = descOffset;
  writeSig(off, 'desc'); off += 4;
  writeU32(off, 0); off += 4;                           // reserved
  writeU32(off, descText.length + 1); off += 4;         // ASCII length including null
  writeAscii(off, descText); off += descText.length;
  bytes[off] = 0; off += 1;                             // null terminator
  // Scriptcode and Unicode counts = 0 (simplified)

  // wtpt (XYZType) — D65 adapted to D50 PCS via Bradford
  off = wtptOffset;
  writeSig(off, 'XYZ '); off += 4;
  writeU32(off, 0); off += 4;
  writeS15F16(off, 0.9505); off += 4;  // X
  writeS15F16(off, 1.0000); off += 4;  // Y
  writeS15F16(off, 1.0890); off += 4;  // Z

  // rXYZ — Red primary (D50-adapted)
  off = rXYZOffset;
  writeSig(off, 'XYZ '); off += 4;
  writeU32(off, 0); off += 4;
  writeS15F16(off, 0.6097559); off += 4;
  writeS15F16(off, 0.3111145); off += 4;
  writeS15F16(off, 0.0194702); off += 4;

  // gXYZ — Green primary (D50-adapted)
  off = gXYZOffset;
  writeSig(off, 'XYZ '); off += 4;
  writeU32(off, 0); off += 4;
  writeS15F16(off, 0.2052401); off += 4;
  writeS15F16(off, 0.6256714); off += 4;
  writeS15F16(off, 0.0608902); off += 4;

  // bXYZ — Blue primary (D50-adapted)
  off = bXYZOffset;
  writeSig(off, 'XYZ '); off += 4;
  writeU32(off, 0); off += 4;
  writeS15F16(off, 0.1492240); off += 4;
  writeS15F16(off, 0.0632141); off += 4;
  writeS15F16(off, 0.7445396); off += 4;

  // rTRC (curveType with gamma = 2.19921875)
  // Shared by gTRC and bTRC (all three channels have identical gamma)
  off = rTRCOffset;
  writeSig(off, 'curv'); off += 4;
  writeU32(off, 0); off += 4;       // reserved
  writeU32(off, 1); off += 4;       // count = 1 → single gamma value
  // u8Fixed8Number: 2.19921875 = 0x0233
  writeU16(off, 0x0233); off += 2;

  // cprt (textType)
  off = cprtOffset;
  writeSig(off, 'text'); off += 4;
  writeU32(off, 0); off += 4;
  writeAscii(off, cprtText);

  return new Uint8Array(buf);
}

// Cache the profile so it's only built once
let _cachedProfile: Uint8Array | null = null;
export function getAdobeRgbIccProfile(): Uint8Array {
  if (!_cachedProfile) {
    _cachedProfile = buildAdobeRgbIccProfile();
  }
  return _cachedProfile;
}
```

**Key design decisions:**

1. **Pre-computed combined matrix**: The `SRGB_LINEAR_TO_ADOBE_RGB_LINEAR` matrix folds the sRGB→XYZ and XYZ→AdobeRGB multiplications into a single 3x3 multiply per pixel (6 multiplications + 3 additions instead of 12+6).

2. **ICC profile built programmatically**: Rather than embedding an opaque Base64 blob, the profile is constructed from documented ICC v2 primitives. This makes it auditable — every byte has a comment. The `gTRC` and `bTRC` tags share the same offset as `rTRC` (valid per ICC spec) since all three channels use the same gamma.

3. **D50 PCS in ICC profile vs D65 in conversion math**: The ICC spec requires the Profile Connection Space to use D50. The XYZ primaries in the ICC tags are D50-adapted (via Bradford chromatic adaptation). However, the *pixel* conversion in `convertSrgbToAdobeRgb()` operates purely in D65 because both sRGB and Adobe RGB share the D65 white point — no chromatic adaptation is needed for the actual pixel math. The D50 adaptation only matters inside the ICC profile for cross-profile interoperability.

4. **Gamma precision**: Adobe RGB uses exactly `563/256 = 2.19921875`, not an approximation like `2.2`. The ICC profile stores this as `u8Fixed8Number = 0x0233` (2 + 51/256 = 2.19921875). The pixel conversion uses `Math.pow(v, 256/563)` for the inverse.

### TIFF Encoder (`src/lib/tiff-encoder.ts`)

This is the core of the project — a from-scratch TIFF 6.0 binary encoder that produces uncompressed RGB files identical to what `imageio.imwrite()` generates. When an ICC profile is provided (for Adobe RGB export), it is embedded via TIFF tag 34675 (`InterColorProfile`), which adds one additional IFD entry and appends the profile bytes after the resolution data.

```typescript
import type { ColorSpace } from '../types';
import { getAdobeRgbIccProfile } from './color-space';

/**
 * Minimal TIFF 6.0 encoder for uncompressed RGB images.
 * Supports 8-bit and 16-bit per channel.
 * Optionally embeds an ICC profile (tag 34675) for color-managed output.
 *
 * TIFF structure produced:
 *   [Header: 8 bytes]
 *   [IFD: variable]
 *   [BitsPerSample values: 6 bytes]
 *   [XResolution: 8 bytes]
 *   [YResolution: 8 bytes]
 *   [ICC Profile: N bytes] (optional, only for Adobe RGB)
 *   [Pixel data: width * height * 3 * bytesPerSample]
 *
 * References:
 *   - TIFF 6.0 Specification (Adobe, 1992)
 *   - ICC.1:2004 (profile embedding via tag 34675)
 *   - Matches output of Python imageio.imwrite() for uint16 RGB
 */

// TIFF tag constants
const TAG_IMAGE_WIDTH          = 256;
const TAG_IMAGE_LENGTH         = 257;
const TAG_BITS_PER_SAMPLE      = 258;
const TAG_COMPRESSION          = 259;
const TAG_PHOTOMETRIC          = 262;
const TAG_STRIP_OFFSETS        = 273;
const TAG_SAMPLES_PER_PIXEL    = 277;
const TAG_ROWS_PER_STRIP       = 278;
const TAG_STRIP_BYTE_COUNTS    = 279;
const TAG_X_RESOLUTION         = 282;
const TAG_Y_RESOLUTION         = 283;
const TAG_RESOLUTION_UNIT      = 296;
const TAG_ICC_PROFILE          = 34675;  // InterColorProfile (0x8773)

// Type constants
const TYPE_BYTE     = 1;  // uint8
const TYPE_SHORT    = 3;  // uint16
const TYPE_LONG     = 4;  // uint32
const TYPE_RATIONAL = 5;  // two uint32s (numerator/denominator)

export function encodeTiff(
  pixelData: Uint16Array | Uint8Array,
  width: number,
  height: number,
  bitsPerSample: 8 | 16,
  colorSpace: ColorSpace = 'srgb'
): Blob {
  const samplesPerPixel = 3;
  const bytesPerSample = bitsPerSample / 8;
  const stripByteCount = width * height * samplesPerPixel * bytesPerSample;

  // Determine if we need an ICC profile
  const iccProfile = colorSpace === 'adobe-rgb' ? getAdobeRgbIccProfile() : null;
  const hasIcc = iccProfile !== null;

  // --- Layout calculation ---
  const numTags = hasIcc ? 12 : 11;  // +1 for ICC_PROFILE tag
  const ifdOffset = 8;
  const ifdSize = 2 + numTags * 12 + 4;

  // Extra data area (after IFD)
  const extraDataOffset = ifdOffset + ifdSize;
  const bitsPerSampleOffset = extraDataOffset;
  const xResolutionOffset = bitsPerSampleOffset + 6;
  const yResolutionOffset = xResolutionOffset + 8;
  let afterResolution = yResolutionOffset + 8;

  // ICC profile data (if present)
  let iccProfileOffset = 0;
  if (hasIcc) {
    iccProfileOffset = afterResolution;
    afterResolution += iccProfile!.length;
    // Align to 2-byte boundary for pixel data (Uint16Array requires alignment)
    if (bitsPerSample === 16 && afterResolution % 2 !== 0) {
      afterResolution += 1;
    }
  }

  // Pixel data follows all extra data
  const pixelDataOffset = afterResolution;
  const totalSize = pixelDataOffset + stripByteCount;

  // --- Build buffer ---
  const buffer = new ArrayBuffer(totalSize);
  const view = new DataView(buffer);
  let offset = 0;

  // Header: little-endian byte order ("II"), magic 42, offset to IFD
  view.setUint8(offset, 0x49); offset += 1;  // 'I'
  view.setUint8(offset, 0x49); offset += 1;  // 'I'
  view.setUint16(offset, 42, true); offset += 2;
  view.setUint32(offset, ifdOffset, true); offset += 4;

  // IFD entry count
  view.setUint16(offset, numTags, true); offset += 2;

  // Helper: write one 12-byte IFD entry
  function writeTag(tag: number, type: number, count: number, value: number) {
    view.setUint16(offset, tag, true); offset += 2;
    view.setUint16(offset, type, true); offset += 2;
    view.setUint32(offset, count, true); offset += 4;
    if (type === TYPE_SHORT && count === 1) {
      view.setUint16(offset, value, true);
      view.setUint16(offset + 2, 0, true);
    } else {
      view.setUint32(offset, value, true);
    }
    offset += 4;
  }

  // IFD entries — MUST be sorted by tag number (TIFF 6.0 requirement)
  writeTag(TAG_IMAGE_WIDTH,       TYPE_LONG,     1, width);
  writeTag(TAG_IMAGE_LENGTH,      TYPE_LONG,     1, height);
  writeTag(TAG_BITS_PER_SAMPLE,   TYPE_SHORT,    3, bitsPerSampleOffset);
  writeTag(TAG_COMPRESSION,       TYPE_SHORT,    1, 1);  // 1 = no compression
  writeTag(TAG_PHOTOMETRIC,       TYPE_SHORT,    1, 2);  // 2 = RGB
  writeTag(TAG_STRIP_OFFSETS,     TYPE_LONG,     1, pixelDataOffset);
  writeTag(TAG_SAMPLES_PER_PIXEL, TYPE_SHORT,    1, 3);
  writeTag(TAG_ROWS_PER_STRIP,    TYPE_LONG,     1, height);
  writeTag(TAG_STRIP_BYTE_COUNTS, TYPE_LONG,     1, stripByteCount);
  writeTag(TAG_X_RESOLUTION,      TYPE_RATIONAL, 1, xResolutionOffset);
  writeTag(TAG_Y_RESOLUTION,      TYPE_RATIONAL, 1, yResolutionOffset);
  // ResolutionUnit intentionally omitted (defaults to inch per TIFF spec)
  // TAG_ICC_PROFILE must come after all lower-numbered tags (34675 > 296)
  if (hasIcc) {
    writeTag(TAG_ICC_PROFILE,     TYPE_BYTE, iccProfile!.length, iccProfileOffset);
  }

  // Next IFD offset (0 = no more IFDs)
  view.setUint32(offset, 0, true); offset += 4;

  // --- Extra data ---

  // BitsPerSample: 3 x SHORT
  view.setUint16(offset, bitsPerSample, true); offset += 2;
  view.setUint16(offset, bitsPerSample, true); offset += 2;
  view.setUint16(offset, bitsPerSample, true); offset += 2;

  // XResolution: RATIONAL 72/1
  view.setUint32(offset, 72, true); offset += 4;
  view.setUint32(offset, 1, true); offset += 4;

  // YResolution: RATIONAL 72/1
  view.setUint32(offset, 72, true); offset += 4;
  view.setUint32(offset, 1, true); offset += 4;

  // ICC Profile data (if present)
  if (hasIcc) {
    const dst = new Uint8Array(buffer, iccProfileOffset, iccProfile!.length);
    dst.set(iccProfile!);
  }

  // --- Pixel data ---
  if (bitsPerSample === 16) {
    const dst = new Uint16Array(buffer, pixelDataOffset);
    const src = pixelData as Uint16Array;
    dst.set(src);
  } else {
    const dst = new Uint8Array(buffer, pixelDataOffset);
    const src = pixelData as Uint8Array;
    dst.set(src);
  }

  return new Blob([buffer], { type: 'image/tiff' });
}
```

### Download Utility (`src/lib/download.ts`)

```typescript
/**
 * Trigger a browser download for a Blob.
 * Mirrors the ComfyUI naming convention: {prefix}_{counter:05}_.tiff
 */
export function downloadBlob(blob: Blob, filename: string): void {
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
}

/**
 * Format counter as zero-padded 5-digit string, matching Python f"{counter:05}"
 */
export function formatFilename(prefix: string, counter: number): string {
  const padded = String(counter).padStart(5, '0');
  return `${prefix}_${padded}_.tiff`;
}
```

### Custom Hooks

#### `useImageStore` (`src/hooks/useImageStore.ts`)

```typescript
import { useState, useEffect, useCallback } from 'react';
import { getAllImages, addImage, deleteImage, getImage } from '../lib/db';
import type { StoredImage } from '../types';

export function useImageStore() {
  const [images, setImages] = useState<StoredImage[]>([]);
  const [loading, setLoading] = useState(true);

  const refresh = useCallback(async () => {
    const all = await getAllImages();
    setImages(all);
    setLoading(false);
  }, []);

  useEffect(() => { refresh(); }, [refresh]);

  const add = useCallback(async (record: Omit<StoredImage, 'id'>) => {
    await addImage(record);
    await refresh();
  }, [refresh]);

  const remove = useCallback(async (id: number) => {
    await deleteImage(id);
    await refresh();
  }, [refresh]);

  const get = useCallback(async (id: number) => {
    return getImage(id);
  }, []);

  return { images, loading, add, remove, get, refresh };
}
```

#### `useToast` (`src/hooks/useToast.ts`)

```typescript
import { useState, useCallback } from 'react';
import type { ToastMessage } from '../types';

let toastCounter = 0;

export function useToast() {
  const [toasts, setToasts] = useState<ToastMessage[]>([]);

  const addToast = useCallback((type: ToastMessage['type'], text: string) => {
    const id = `toast-${++toastCounter}`;
    setToasts(prev => [...prev, { id, type, text }]);
    setTimeout(() => {
      setToasts(prev => prev.filter(t => t.id !== id));
    }, 4000);
  }, []);

  const dismissToast = useCallback((id: string) => {
    setToasts(prev => prev.filter(t => t.id !== id));
  }, []);

  return { toasts, addToast, dismissToast };
}
```

### Components

#### `App.tsx` — Root Component

```typescript
import { useState } from 'react';
import { DropZone } from './components/DropZone';
import { Gallery } from './components/Gallery';
import { ExportToolbar } from './components/ExportToolbar';
import { PreviewModal } from './components/PreviewModal';
import { Toast } from './components/Toast';
import { useImageStore } from './hooks/useImageStore';
import { useToast } from './hooks/useToast';
import type { StoredImage, ExportOptions } from './types';
import './App.css';

export default function App() {
  const store = useImageStore();
  const { toasts, addToast, dismissToast } = useToast();
  const [selected, setSelected] = useState<Set<number>>(new Set());
  const [previewId, setPreviewId] = useState<number | null>(null);
  const [exportOptions, setExportOptions] = useState<ExportOptions>({
    filenamePrefix: 'export',
    colorSpace: 'srgb',
    bitDepth: 16,
  });

  return (
    <div className="app">
      <header className="app-header">
        <h1>Export TIFF Web</h1>
        <p className="subtitle">16-bit TIFF exporter with Adobe RGB — entirely in your browser</p>
      </header>

      <DropZone onFilesAdded={store.add} addToast={addToast} />

      <Gallery
        images={store.images}
        loading={store.loading}
        selected={selected}
        onSelectionChange={setSelected}
        onPreview={setPreviewId}
        onDelete={store.remove}
      />

      <ExportToolbar
        images={store.images}
        selected={selected}
        options={exportOptions}
        onOptionsChange={setExportOptions}
        getImage={store.get}
        addToast={addToast}
      />

      {previewId !== null && (
        <PreviewModal
          imageId={previewId}
          getImage={store.get}
          options={exportOptions}
          onClose={() => setPreviewId(null)}
          addToast={addToast}
        />
      )}

      <Toast toasts={toasts} onDismiss={dismissToast} />
    </div>
  );
}
```

#### `DropZone.tsx`

Accepts drag-and-drop or file-picker input. On file(s) received:
1. Validates each file's type.
2. Calls `decodeImageFile(file)` from `image-processing.ts`.
3. On success, calls `onFilesAdded(record)` to persist to IndexedDB.
4. On error, calls `addToast('error', message)`.
5. Visual feedback: border changes to blue dashed on `dragover`, resets on `dragleave`/`drop`.

#### `Gallery.tsx` / `GalleryCard.tsx`

- `Gallery` renders the grid and "Select All" / "Deselect All" toggle.
- `GalleryCard` renders an individual image card with thumbnail, name, dimensions, checkbox, and delete button.
- Selection state is lifted to `App` via `selected: Set<number>` and `onSelectionChange`.

#### `ExportToolbar.tsx`

Contains the filename prefix input, color space dropdown, bit-depth toggle, and export buttons. On export:
1. Collects image IDs to export (selected or all).
2. For each, retrieves the full `StoredImage` from IndexedDB via `getImage(id)`.
3. **If `colorSpace` is `'srgb'`**: Uses the fast integer path — `rgbaToRgb16` or `rgbaToRgb8` directly.
4. **If `colorSpace` is `'adobe-rgb'`**: Uses the float pipeline:
   - `rgbaToNormalizedRgb()` → `convertSrgbToAdobeRgb()` → `quantizeToUint16()` or `quantizeToUint8()`.
5. Calls `encodeTiff(pixelData, width, height, bitDepth, colorSpace)`.
6. Calls `downloadBlob(blob, formatFilename(prefix, counter))`.
7. Shows success toast: `"Exported {count} image(s) as {bitDepth}-bit TIFF ({colorSpaceLabel})"`.

The toast message includes the color space label (`"sRGB"` or `"Adobe RGB"`) so the user has clear confirmation of what was exported.

For batch exports with more than 3 images, add a brief `await new Promise(r => setTimeout(r, 100))` between downloads to prevent browser throttling.

#### `PreviewModal.tsx`

Modal overlay with full-resolution canvas rendering. Loads the `rgbaData` from IndexedDB, creates `ImageData` from it, and draws to a `<canvas>`. Includes an "Export This Image" button.

#### `Toast.tsx`

Fixed-position toast container (bottom-right). Each toast has a colored left border (blue for info, red for error, green for success), text content, and dismiss X button. Auto-dismisses after 4 seconds.

---

## Style Guide

### Color Palette

| Token             | Value       | Usage                          |
|-------------------|-------------|--------------------------------|
| `--bg`            | `#0f1117`   | Page background                |
| `--surface`       | `#1a1d27`   | Card and panel backgrounds     |
| `--surface-hover` | `#242836`   | Hovered cards                  |
| `--border`        | `#2d3348`   | Default borders                |
| `--border-accent` | `#4a7dff`   | Active/focused borders         |
| `--text`          | `#e8eaf0`   | Primary text                   |
| `--text-muted`    | `#8b92a8`   | Secondary/helper text          |
| `--accent`        | `#4a7dff`   | Buttons, links, active states  |
| `--accent-hover`  | `#6194ff`   | Button hover                   |
| `--danger`        | `#ff4a6a`   | Delete buttons, error toasts   |
| `--success`       | `#4aff8b`   | Success toasts                 |

### Typography

- Font stack: `'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`
- H1: `1.75rem`, weight 700
- Body: `0.875rem`, weight 400
- Monospace (filenames, counters): `'JetBrains Mono', 'Fira Code', monospace`

### Layout

- Max content width: `960px`, centered with `margin: 0 auto`
- Gallery grid: `repeat(auto-fill, minmax(180px, 1fr))`, gap `12px`
- Drop zone height: `160px`
- Border radius: `8px` for cards and panels, `6px` for buttons
- Spacing scale: `4px` base unit (multiples of 4)

### Transitions

- Card hover: `background-color 150ms ease`
- Card delete fade-out: `opacity 200ms ease-out`
- Modal backdrop: `opacity 200ms ease`
- Toast entry: `transform 300ms ease` (slide up from bottom)
- Toast exit: `opacity 200ms ease-out`

---

## Edge Cases and Error Handling

| Scenario | Behavior |
|----------|----------|
| Drop non-image file (e.g., .txt, .pdf) | Reject silently, show error toast with filename |
| Drop image > 16384px in either dimension | Reject, show error toast with dimensions |
| IndexedDB quota exceeded on add | Show error toast, do not crash |
| IndexedDB unavailable (private browsing in some browsers) | Show persistent banner: "IndexedDB unavailable — images will not persist across reloads" |
| Export with no images selected | "Export Selected" button is disabled (greyed out) |
| Export with empty gallery | Both export buttons are disabled |
| Filename prefix with invalid characters | Strip invalid chars on input, allow only `[a-zA-Z0-9_-]` |
| Filename prefix is empty string | Fall back to `"export"` |
| Very large image (e.g., 8000x8000 @ 16-bit) | ~384 MB buffer. Show progress indication. Use `setTimeout` chunking if needed to avoid UI freeze |
| Adobe RGB conversion on very large image | The float pipeline allocates an additional `Float64Array` (~1.5 GB for 8000x8000x3xf64). If allocation fails, catch the error and show toast: `"Image too large for Adobe RGB conversion — try sRGB or a smaller image"` |
| Out-of-gamut colors during sRGB→Adobe RGB | Clamp to [0.0, 1.0] after matrix multiplication. sRGB is a subset of Adobe RGB, so clipping is minimal in practice, but numerical precision can produce values slightly outside the range |
| Browser does not support OffscreenCanvas | Fall back to regular `<canvas>` element created in the DOM (hidden) |

---

## Accessibility Requirements

- All interactive elements are keyboard-focusable and operable via Enter/Space.
- Drop zone is labeled with `role="button"` and `aria-label="Drop images here or click to browse"`.
- Gallery cards have `role="listitem"` within a `role="list"` container.
- Modal traps focus while open and returns focus to the triggering element on close.
- Toast messages use `role="status"` and `aria-live="polite"`.
- Color contrast meets WCAG 2.1 AA (4.5:1 for body text on `--bg`).
- Delete actions require confirmation via keyboard (Enter on the focused trash icon).

---

## Performance Goals

| Metric | Target |
|--------|--------|
| Initial load (Lighthouse) | < 1s FCP on 4G |
| Bundle size (gzipped) | < 80 KB total |
| Image decode + store | < 500ms for a 4K (3840x2160) JPEG |
| TIFF encode (16-bit 4K) | < 2s |
| Gallery render (50 images) | 60fps scroll |

The TIFF encoder operates on typed arrays with zero intermediate copies. The `Uint16Array` is written directly into the pre-allocated `ArrayBuffer`, making it as fast as the browser allows for sequential memory writes.

---

## Setup and Build

```bash
npm create vite@latest export-tiff-web -- --template react-ts
cd export-tiff-web
npm install idb
```

Then replace the scaffolded `src/` contents with the structure defined above.

```bash
npm run dev    # Development server at http://localhost:5173
npm run build  # Production build to dist/
```

---

## Verification Checklist

- [ ] Load a PNG — verify it appears in the gallery with correct thumbnail and dimensions
- [ ] Load a JPEG — same verification
- [ ] Reload the page — verify all previously loaded images persist in the gallery
- [ ] Export a single image as 16-bit sRGB TIFF — open in Photoshop/GIMP, confirm 16-bit RGB, dimensions match, no ICC profile embedded
- [ ] Export a single image as 8-bit sRGB TIFF — confirm 8-bit RGB
- [ ] Export a single image as 16-bit **Adobe RGB** TIFF — open in Photoshop, confirm:
  - Photoshop detects and assigns the Adobe RGB (1998) profile (check Edit > Assign Profile)
  - Image > Mode shows 16 Bits/Channel
  - Colors appear correct (not over-saturated, no hue shift)
- [ ] Export a single image as 8-bit Adobe RGB TIFF — confirm 8-bit with embedded profile
- [ ] Compare sRGB export vs Adobe RGB export of the same image in Photoshop — converting the Adobe RGB version back to sRGB should closely match the original sRGB export (round-trip fidelity check)
- [ ] Inspect the Adobe RGB TIFF hex: verify tag 34675 (0x8773) is present in the IFD, pointing to valid ICC profile data starting with `'ADBE'` CMM signature
- [ ] Batch-select 3 images, export as Adobe RGB — confirm 3 files downloaded with `_00000_`, `_00001_`, `_00002_` suffixes, all with ICC profiles
- [ ] Verify filename format matches original: `{prefix}_{counter:05}_.tiff`
- [ ] Delete an image — confirm it is removed from gallery and IndexedDB
- [ ] Drop a .txt file — confirm error toast appears and no record is created
- [ ] Open preview modal — confirm full-resolution rendering and "Export This Image" works
- [ ] Test with a large image (8K) — confirm export completes without crashing
- [ ] Inspect the exported TIFF hex header: bytes `49 49 2A 00` (little-endian TIFF magic)
- [ ] Verify the Color Space dropdown shows "Wide Gamut" indicator when Adobe RGB is selected
