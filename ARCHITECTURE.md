# Map Revealer

A two-page HTML/CSS/JavaScript application for masking regions of an image in an editor window and transmitting the unmasked portion to a separate preview window.

## Architecture

| File | Role |
| --- | --- |
| `index.html` | Editor. Holds the full-resolution image, a binary mask, and an annotation layer. |
| `preview.html` | Receiver. Renders only the data pushed to it by the editor. |

Inter-window communication uses `window.postMessage` with a wildcard target origin. No server, no backend, no build step, no external dependencies.

### Editor canvas stack

Three `<canvas>` elements stacked in a container, all sized to the image's natural dimensions. Pan/zoom is applied to the container via CSS `transform`, so the three layers move in lockstep without per-canvas redraws.

| Layer | Purpose |
| --- | --- |
| `imageCanvas` | The loaded image, drawn once. |
| `maskCanvas` | Overlay rendered from `maskData`, a `Uint8Array` of length `width * height` (`0` = hidden, `1` = revealed). Repaints only the dirty rectangle per brush stamp. |
| `annotationCanvas` | Freehand strokes from the pen and eraser tools. |

### Data transmission

The editor builds an offscreen canvas with the image and alpha-zeroes every pixel where `maskData === 0`, then converts both that canvas and the annotation canvas to `ImageBitmap` via `createImageBitmap()` and posts them as **transferables** to the preview window. Transfer is zero-copy — no base64, no PNG round-trip. The preview draws each bitmap onto its canvas and calls `.close()` to release it immediately.

A `ready` handshake replaces ad-hoc delays: when `preview.html` loads it posts `{type: 'ready'}` to `window.opener`; the editor only sends after receiving it. Pending messages from a Send issued before the preview is ready are queued and flushed on `ready`.

### Auto-sync

A toggle in the toolbar enables auto-sync: any modification (brush stroke, rectangle, reveal/hide all, undo/redo, pen/eraser stroke) schedules a debounced send (350 ms) to the preview. The toggle state is persisted in `localStorage` under `mapeditor_autosync`. Manual *Send* still works and clears any pending auto-sync timer.

### Undo / redo

A unified history covers both mask edits and annotation strokes. Each entry is one of:
- `{type: 'mask', mask: Uint8Array}` — a snapshot of `maskData`.
- `{type: 'anno', imgData: ImageData, blob: Blob}` — annotation pixels. The `ImageData` is captured synchronously, then `canvas.toBlob('image/png')` runs asynchronously and replaces it with a much smaller PNG `Blob` to free RAM. Restore prefers `imgData` if still present, otherwise decodes the blob via `createImageBitmap`.

The history is bounded both by count (`MAX_HISTORY = 50`) and by total bytes across history+redo (`MAX_HISTORY_BYTES = 256 MB`); when either is exceeded, the oldest entries are dropped.

### Project save / load (`.mrp`)

The Save button serializes a JSON blob with three embedded PNG data URLs (image, mask, annotations) plus dimensions and writes it as `<basename>.mrp`. Open reverses the process: it fetches all three PNGs, repaints the canvases, decodes the mask PNG's red channel back into `maskData`, and resets history. Project state is fully self-contained — no reference to the source image file.

The mask PNG is grayscale (`R = maskData[i] * 255`), which compresses well thanks to the large uniform regions typical of map masks.

### Large-image resize prompt

If a loaded image exceeds `MAX_IMAGE_DIM = 4096` per side or `MAX_IMAGE_PIXELS = 16 MP`, the editor opens a modal proposing scaled-down dimensions (preserving aspect ratio, the stricter constraint wins). The user can accept (`drawImage` to a smaller canvas, `imageSmoothingQuality = 'high'`) or keep the original.

## Usage

1. Open `index.html` in a browser. Double-clicking the file is sufficient; the application runs from `file://`.
2. Click *Load* (or drag-and-drop a file onto the viewport, or press `Ctrl+O`). The editor initializes with a red overlay covering the entire image and automatically opens `preview.html` in a new window. The current file name appears next to the load button.
3. Paint with the *Select* brush (S) to mark areas as revealed (green overlay replaces red); *Deselect* (D) to restore the red overlay; *Rectangle* (R) to fill rectangular regions in one drag — hold `Alt` to deselect with the rectangle. Use *Pen* (P) and *Eraser* (E) to manage annotations in any color.
4. Click *Send* (or press `Ctrl+S`) to push the current revealed image and annotations to the preview window. Or enable the *Auto* toggle for live updates.
5. Use *Export PNG* to download the flattened result (revealed image + annotations) as a single PNG.
6. Use *Save project* / *Open project* to persist and resume the full editing state in a `.mrp` file.

### Toolbar groups

The toolbar is divided into five groups separated by thin vertical dividers, with the output group pushed to the right:

| Group | Contents |
| --- | --- |
| File / project | Load image, file name, Open project, Save project |
| Tools | Cursor, Select, Deselect, Rectangle, Pen, Eraser |
| Mask & history | Reveal all, Hide all, Undo, Redo |
| Options | Pen color, Brush size |
| Output (right) | Export PNG, Auto-sync, Send |

A language selector sits at the far right after a final divider.

### Multiple instances

Each editor tab opens its preview with a unique window name (timestamp-based), so `index.html` can be opened in any number of tabs simultaneously. Each editor communicates only with its own preview window; instances do not interfere with one another.

## Running locally

Opening `index.html` directly from the filesystem works in Chromium-based browsers, Firefox, and Safari. The `postMessage` call uses `'*'` as the target origin, which is permitted between same-origin `file://` windows.

If the host environment restricts script execution from `file://`, serve the directory over HTTP:

```bash
python -m http.server 8000
# or
npx serve .
# or VS Code's Live Server extension
```

## Keyboard shortcuts

### Editor

| Shortcut | Action |
| --- | --- |
| `V` / `1` | Cursor (pan and zoom) |
| `S` / `2` | Select brush (reveal) |
| `D` / `3` | Deselect brush (hide) |
| `P` / `4` | Pen |
| `E` / `5` | Eraser |
| `R` / `6` | Rectangle (hold `Alt` while releasing to deselect) |
| `[` / `]` | Decrease / increase brush size |
| `Space` (hold) | Temporary pan; releases back to previous tool |
| `Esc` | Switch to cursor tool |
| `Ctrl` + `Z` | Undo (mask or annotation, whichever was last) |
| `Ctrl` + `Y` or `Ctrl` + `Shift` + `Z` | Redo |
| `Ctrl` + `S` | Send to preview |
| `Ctrl` + `O` | Open image file |
| Mouse wheel | Adjust brush size (when a brush tool is active) |
| `Ctrl` + wheel | Zoom |

### Preview

| Shortcut | Action |
| --- | --- |
| `F` | Toggle fullscreen |
| Double-click on image | Toggle fullscreen |
| `Esc` | Exit fullscreen (browser default) |

## Localization

UI strings (including button tooltips via `data-i18n-title`) are available in 16 languages: Italian, English, Spanish, French, German, Portuguese, Russian, Japanese, Chinese, Korean, Arabic, Hindi, Polish, Dutch, Turkish, Swedish. The selected language is persisted in `localStorage` under `mapeditor_lang` and propagates to the preview window via the `storage` event.

The native file-picker control is hidden in favor of a custom button with a translated label, so the only browser-controlled text remaining is the OS file dialog itself.

## Browser requirements

Canvas 2D, Pointer Events, `window.postMessage`, `localStorage`, the Fullscreen API, `createImageBitmap` with transferable support, and `canvas.toBlob`.

## Project structure

```
.
├── index.html      Editor (inline CSS + JS)
├── preview.html    Preview (inline CSS + JS)
└── README.md
```
