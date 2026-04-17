# Map Revealer

Two-window tool for revealing pieces of a map (or any image) one bit at a time. The editor is where you paint over what should stay hidden or become visible; the preview window shows only the revealed portion, ready to be put on a second screen for someone else to look at.

Built for tabletop role-playing sessions where the GM wants to gradually expose a map to the players, but it works for anything where you want to control what part of an image another person sees.

## How it works

Open `index.html` in any modern browser. Load an image — the editor covers it with a red overlay (everything is hidden) and opens a second window for the preview.

From there:

- **Select** brush paints green on the areas the players should see.
- **Deselect** brush goes back to red.
- **Rectangle** does the same as a single drag — hold `Alt` to deselect with it.
- **Pen** and **Eraser** add freehand annotations on top, in any color.
- **Reveal all** / **Hide all** flip the whole map at once.
- Pan with the cursor tool or by holding `Space`. Zoom with `Ctrl`+wheel.

When you're ready, click **Send** (or `Ctrl+S`) and the preview updates with the revealed image plus annotations. Or turn on **Auto** and the preview updates by itself a fraction of a second after every change.

## Saving your work

- **Export PNG** writes the current revealed view (image + annotations, holes where the mask is) as a flat PNG.
- **Save project** / **Open project** save and reopen a `.mrp` file containing the original image, the mask, and your annotations, so you can pick up exactly where you left off in another session.

## Shortcuts

| | |
| --- | --- |
| `V` `1` | Cursor |
| `S` `2` | Select |
| `D` `3` | Deselect |
| `R` `6` | Rectangle (`Alt` = deselect) |
| `P` `4` | Pen |
| `E` `5` | Eraser |
| `[` `]` | Brush size |
| `Space` | Hold for temporary pan |
| `Esc` | Back to cursor |
| `Ctrl`+`Z` / `Ctrl`+`Y` | Undo / redo |
| `Ctrl`+`S` | Send to preview |
| `Ctrl`+`O` | Open image |
| Wheel | Brush size (or zoom with `Ctrl`) |

In the preview: `F` or double-click for fullscreen.

## Languages

UI in 16 languages. The selector is at the top-right; the choice is remembered between sessions and applies to the preview window too.

## Running it

The application is two static HTML files and runs from `file://` — double-click `index.html` and you're done. If your browser refuses to run scripts from disk, serve the folder over HTTP:

```bash
python -m http.server 8000
```

then open `http://localhost:8000`.

## Project files

```
.
├── index.html        Editor
├── preview.html      Preview window
├── README.md         This file
└── ARCHITECTURE.md   Technical notes (canvas layout, message protocol, history, file format)
```
