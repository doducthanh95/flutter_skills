---
name: excalidraw
description: Create hand-drawn style diagrams using Excalidraw JSON format. Generate .excalidraw files for architecture diagrams, flowcharts, sequence diagrams, concept maps, and more. Files can be opened at excalidraw.com or uploaded for shareable links.
version: 1.0.0
author: Hermes Agent
license: MIT
dependencies: []
metadata:
  hermes:
    tags: [Excalidraw, Diagrams, Flowcharts, Architecture, Visualization, JSON]
    related_skills: []

---

# Excalidraw Diagram Skill

Create diagrams by writing standard Excalidraw element JSON and saving as `.excalidraw` files. These files can be drag-and-dropped onto [excalidraw.com](https://excalidraw.com) for viewing and editing. No accounts, no API keys, no rendering libraries -- just JSON.

## Workflow

1. **Load this skill** (you already did)
2. **Write the elements JSON** -- an array of Excalidraw element objects
3. **Save the file** using `write_file` to create a `.excalidraw` file
4. **Optionally upload** for a shareable link using `scripts/upload.py` via `terminal`

### Saving a Diagram

Wrap your elements array in the standard `.excalidraw` envelope and save with `write_file`:

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "hermes-agent",
  "elements": [ ...your elements array here... ],
  "appState": {
    "viewBackgroundColor": "#ffffff"
  }
}
```

Save to any path, e.g. `~/diagrams/my_diagram.excalidraw`.

### Uploading for a Shareable Link

Run the upload script (located in this skill's `scripts/` directory) via terminal:

```bash
python skills/diagramming/excalidraw/scripts/upload.py ~/diagrams/my_diagram.excalidraw
```

This uploads to excalidraw.com (no account needed) and prints a shareable URL. Requires the `cryptography` pip package (`pip install cryptography`).

### Rendering & exporting locally (PNG/SVG)

Notes from practice: when generating `.excalidraw` JSON programmatically, you may want to export local PNG/SVG without opening the web UI. Common pitfalls and a reusable approach:

- Validate JSON: Ensure the `.excalidraw` file is valid JSON (no stray quotes or concatenated documents). If JSON parsing fails, check for accidental string concatenation or malformed fields such as `"updated":168"` (extra quote).
- Fonts: Pillow needs an available TTF to render text. Try common paths like `/Library/Fonts/Arial.ttf` or `/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf`.
- Environment: If Pillow (PIL) is missing, install it with `python3 -m pip install --user pillow`.
- Fallback: If Pillow is unavailable or you prefer a vector export, open the `.excalidraw` file at https://excalidraw.com and export PNG/SVG there.

Reusable Python script (minimal) to render a crude PNG from an `.excalidraw` file using Pillow (adjust paths/fonts as needed):

```python
# render_excalidraw_png.py
import json
from PIL import Image, ImageDraw, ImageFont

def render(in_path, out_path):
    with open(in_path, 'r', encoding='utf-8') as f:
        doc = json.load(f)
    elements = doc.get('elements', [])
    # compute canvas bounds
    min_x = min((el.get('x',0) for el in elements), default=0)
    min_y = min((el.get('y',0) for el in elements), default=0)
    max_x = max((el.get('x',0)+el.get('width',0) for el in elements), default=800)
    max_y = max((el.get('y',0)+el.get('height',0) for el in elements), default=600)
    margin = 40
    W = max(800, int(max_x - min_x + 2*margin))
    H = max(600, int(max_y - min_y + 2*margin))

    img = Image.new('RGB', (W, H), 'white')
    draw = ImageDraw.Draw(img)

    def parse_color(h):
        if not h: return (255,255,255)
        if isinstance(h,str) and h.startswith('#') and len(h)>=7:
            return tuple(int(h[i:i+2],16) for i in (1,3,5))
        return (255,255,255)

    # simple render loop (rectangle/ellipse/diamond/arrow/text)
    for el in elements:
        etype = el.get('type')
        x = int(el.get('x',0) - min_x + margin)
        y = int(el.get('y',0) - min_y + margin)
        w = int(el.get('width',0))
        h = int(el.get('height',0))
        bbox = (x, y, x+w, y+h)
        bg = parse_color(el.get('backgroundColor'))
        stroke = parse_color(el.get('strokeColor') or '#111')
        if etype == 'rectangle':
            draw.rounded_rectangle(bbox, radius=8, fill=bg, outline=stroke, width=2)
        elif etype == 'ellipse':
            draw.ellipse(bbox, fill=bg, outline=stroke)
        elif etype == 'diamond':
            cx = x + w/2; cy = y + h/2
            pts = [(cx,y),(x+w,cy),(cx,y+h),(x,cy)]
            draw.polygon(pts, fill=bg, outline=stroke)
        elif etype == 'arrow':
            pts = el.get('points', [[0,0],[max(10,w),0]])
            abs_pts = [(x+dx,y+dy) for dx,dy in pts]
            draw.line(abs_pts, fill=stroke, width=2)
            if len(abs_pts)>=2:
                x1,y1 = abs_pts[-1]
                draw.polygon([(x1,y1),(x1-8,y1-6),(x1-8,y1+6)], fill=stroke)
        elif etype == 'text':
            text = el.get('text','')
            font_size = int(el.get('fontSize',16))
            try:
                font = ImageFont.truetype('/Library/Fonts/Arial.ttf', font_size)
            except:
                font = ImageFont.load_default()
            lines = text.split('\n')
            for i,line in enumerate(lines):
                draw.text((x+6, y+6 + i*(font_size+2)), line, fill=(0,0,0), font=font)

    img.save(out_path)

if __name__ == '__main__':
    import sys
    render(sys.argv[1], sys.argv[2])
```

If you find yourself needing to export programmatically often, consider adding this script to `scripts/render_excalidraw.py` in the skill and documenting that Pillow is a runtime dependency.



Run the upload script (located in this skill's `scripts/` directory) via terminal:

```bash
python skills/diagramming/excalidraw/scripts/upload.py ~/diagrams/my_diagram.excalidraw
```

This uploads to excalidraw.com (no account needed) and prints a shareable URL. Requires the `cryptography` pip package (`pip install cryptography`).

---

## Element Format Reference

### Required Fields (all elements)
`type`, `id` (unique string), `x`, `y`, `width`, `height`

### Defaults (skip these -- they're applied automatically)
- `strokeColor`: `"#1e1e1e"`
- `backgroundColor`: `"transparent"`
- `fillStyle`: `"solid"`
- `strokeWidth`: `2`
- `roughness`: `1` (hand-drawn look)
- `opacity`: `100`

Canvas background is white.

### Element Types

**Rectangle**:
```json
{ "type": "rectangle", "id": "r1", "x": 100, "y": 100, "width": 200, "height": 100 }
```
- `roundness: { "type": 3 }` for rounded corners
- `backgroundColor: "#a5d8ff"`, `fillStyle: "solid"` for filled

**Ellipse**:
```json
{ "type": "ellipse", "id": "e1", "x": 100, "y": 100, "width": 150, "height": 150 }
```

**Diamond**:
```json
{ "type": "diamond", "id": "d1", "x": 100, "y": 100, "width": 150, "height": 150 }
```

**Labeled shape (container binding)** -- create a text element bound to the shape:

> **WARNING:** Do NOT use `"label": { "text": "..." }` on shapes. This is NOT a valid
> Excalidraw property and will be silently ignored, producing blank shapes. You MUST
> use the container binding approach below.

The shape needs `boundElements` listing the text, and the text needs `containerId` pointing back:
```json
{ "type": "rectangle", "id": "r1", "x": 100, "y": 100, "width": 200, "height": 80,
  "roundness": { "type": 3 }, "backgroundColor": "#a5d8ff", "fillStyle": "solid",
  "boundElements": [{ "id": "t_r1", "type": "text" }] },
{ "type": "text", "id": "t_r1", "x": 105, "y": 110, "width": 190, "height": 25,
  "text": "Hello", "fontSize": 20, "fontFamily": 1, "strokeColor": "#1e1e1e",
  "textAlign": "center", "verticalAlign": "middle",
  "containerId": "r1", "originalText": "Hello", "autoResize": true }
```
- Works on rectangle, ellipse, diamond
- Text is auto-centered by Excalidraw when `containerId` is set
- The text `x`/`y`/`width`/`height` are approximate -- Excalidraw recalculates them on load
- `originalText` should match `text`
- Always include `fontFamily: 1` (Virgil/hand-drawn font)

**Labeled arrow** -- same container binding approach:
```json
{ "type": "arrow", "id": "a1", "x": 300, "y": 150, "width": 200, "height": 0,
  "points": [[0,0],[200,0]], "endArrowhead": "arrow",
  "boundElements": [{ "id": "t_a1", "type": "text" }] },
{ "type": "text", "id": "t_a1", "x": 370, "y": 130, "width": 60, "height": 20,
  "text": "connects", "fontSize": 16, "fontFamily": 1, "strokeColor": "#1e1e1e",
  "textAlign": "center", "verticalAlign": "middle",
  "containerId": "a1", "originalText": "connects", "autoResize": true }
```

**Standalone text** (titles and annotations only -- no container):
```json
{ "type": "text", "id": "t1", "x": 150, "y": 138, "text": "Hello", "fontSize": 20,
  "fontFamily": 1, "strokeColor": "#1e1e1e", "originalText": "Hello", "autoResize": true }
```
- `x` is the LEFT edge. To center at position `cx`: `x = cx - (text.length * fontSize * 0.5) / 2`
- Do NOT rely on `textAlign` or `width` for positioning

**Arrow**:
```json
{ "type": "arrow", "id": "a1", "x": 300, "y": 150, "width": 200, "height": 0,
  "points": [[0,0],[200,0]], "endArrowhead": "arrow" }
```
- `points`: `[dx, dy]` offsets from element `x`, `y`
- `endArrowhead`: `null` | `"arrow"` | `"bar"` | `"dot"` | `"triangle"`
- `strokeStyle`: `"solid"` (default) | `"dashed"` | `"dotted"`

### Arrow Bindings (connect arrows to shapes)

```json
{
  "type": "arrow", "id": "a1", "x": 300, "y": 150, "width": 150, "height": 0,
  "points": [[0,0],[150,0]], "endArrowhead": "arrow",
  "startBinding": { "elementId": "r1", "fixedPoint": [1, 0.5] },
  "endBinding": { "elementId": "r2", "fixedPoint": [0, 0.5] }
}
```

`fixedPoint` coordinates: `top=[0.5,0]`, `bottom=[0.5,1]`, `left=[0,0.5]`, `right=[1,0.5]`

### Drawing Order (z-order)
- Array order = z-order (first = back, last = front)
- Emit progressively: background zones → shape → its bound text → its arrows → next shape
- BAD: all rectangles, then all texts, then all arrows
- GOOD: bg_zone → shape1 → text_for_shape1 → arrow1 → arrow_label_text → shape2 → text_for_shape2 → ...
- Always place the bound text element immediately after its container shape

### Sizing Guidelines

**Font sizes:**
- Minimum `fontSize`: **16** for body text, labels, descriptions
- Minimum `fontSize`: **20** for titles and headings
- Minimum `fontSize`: **14** for secondary annotations only (sparingly)
- NEVER use `fontSize` below 14

**Element sizes:**
- Minimum shape size: 120x60 for labeled rectangles/ellipses
- Leave 20-30px gaps between elements minimum
- Prefer fewer, larger elements over many tiny ones

### Color Palette

See `references/colors.md` for full color tables. Quick reference:

| Use | Fill Color | Hex |
|-----|-----------|-----|
| Primary / Input | Light Blue | `#a5d8ff` |
| Success / Output | Light Green | `#b2f2bb` |
| Warning / External | Light Orange | `#ffd8a8` |
| Processing / Special | Light Purple | `#d0bfff` |
| Error / Critical | Light Red | `#ffc9c9` |
| Notes / Decisions | Light Yellow | `#fff3bf` |
| Storage / Data | Light Teal | `#c3fae8` |

### Tips
- Use the color palette consistently across the diagram
- **Text contrast is CRITICAL** -- never use light gray on white backgrounds. Minimum text color on white: `#757575`
- Do NOT use emoji in text -- they don't render in Excalidraw's font
- For dark mode diagrams, see `references/dark-mode.md`
- For larger examples, see `references/examples.md`


