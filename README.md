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

When the user triggers *Send*, the editor builds an offscreen canvas, draws the image, sets alpha to `0` for every pixel where `maskData === 0`, serializes the result and the annotation canvas to PNG data URLs, and posts both (plus the dimensions) to the preview window. The preview replaces its canvas contents on every message.

## Usage

1. Open `index.html` in a browser. Double-clicking the file is sufficient; the application runs from `file://`.
2. Load an image via the file input. The editor initializes with a red overlay covering the entire image and automatically opens `preview.html` in a new window.
3. Paint with the *Select* brush to mark areas as revealed (green overlay replaces red). Paint with *Deselect* to restore the red overlay. Use *Pen* and *Eraser* to manage annotations in any color.
4. Click *Send* (or `Ctrl+S`) to push the current revealed image and annotations to the preview window. The preview updates on each send.

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
| `[` / `]` | Decrease / increase brush size |
| `Space` (hold) | Temporary pan; releases back to previous tool |
| `Esc` | Switch to cursor tool |
| `Ctrl` + `Z` | Undo annotation stroke |
| `Ctrl` + `Y` or `Ctrl` + `Shift` + `Z` | Redo annotation stroke |
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
<
## Localization

UI strings are available in 16 languages: Italian, English, Spanish, French, German, Portuguese, Russian, Japanese, Chinese, Korean, Arabic, Hindi, Polish, Dutch, Turkish, Swedish. The selected language is persisted in `localStorage` under `mapeditor_lang` and propagates to the preview window via the `storage` event.

## Browser requirements

Canvas 2D, Pointer Events, `window.postMessage`, `localStorage`, and the Fullscreen API.

## Project structure

```
.
├── index.html      Editor (inline CSS + JS)
├── preview.html    Preview (inline CSS + JS)
└── README.md
```
