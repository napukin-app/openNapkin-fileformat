# `.npkd` File Format Specification

**Version:** 1  
**Application:** Napukin (openNapkin)

---

## 1. File Structure

A `.npkd` file is a **ZIP archive using STORE compression** (no deflation). It contains:

```
├── manifest.json       (required)
├── document.json       (required)
├── thumbnail.png       (optional)
└── assets/             (required, may be empty)
    ├── <id>.png
    ├── <id>.jpg
    └── ...
```

The STORE compression mode means files are stored uncompressed inside the ZIP. This keeps the format simple and makes the JSON contents accessible to tools that can parse ZIP entries without decompression.

The MIME type is `application/zip`. The file extension is `.npkd`.

---

## 2. manifest.json

```json
{
  "app": "Napukin",
  "version": 1,
  "name": "My Design",
  "createdAt": 1711900000000
}
```

| Field | Type | Description |
|-------|------|-------------|
| `app` | string | Always `"Napukin"` |
| `version` | integer | Format version, currently `1` |
| `name` | string | Human-readable document name |
| `createdAt` | integer | Unix timestamp in milliseconds (`Date.now()`) |

---

## 3. document.json

### Top-Level Structure

```json
{
  "name": "My Design",
  "version": 1,
  "artboardWidth": 393,
  "artboardHeight": 852,
  "artboardFill": true,
  "artboardFillColor": "#ffffff",
  "layers": [],
  "comments": [],
  "assetManifest": []
}
```

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Document name (matches manifest) |
| `version` | integer | Format version, currently `1` |
| `artboardWidth` | number | Artboard width in pixels |
| `artboardHeight` | number | Artboard height in pixels |
| `artboardFill` | boolean | Whether the artboard has a background fill (default `true`) |
| `artboardFillColor` | string | Artboard background color as hex string (default `"#ffffff"`) |
| `layers` | array | Ordered list of layer objects (bottom to top) |
| `comments` | array | List of comment objects |
| `assetManifest` | array | Reserved for future use, currently `[]` |

### Standard Artboard Sizes

| Device | Width | Height |
|--------|-------|--------|
| iPhone 17 | 393 | 852 |
| iPhone 17 Pro | 402 | 874 |
| iPad Mini 7 | 744 | 1133 |
| iPad Pro 11" | 834 | 1194 |
| MacBook Air M5 | 1470 | 956 |
| MacBook Pro M5 | 1512 | 982 |
| MacBook Neo | 1280 | 800 |
| FHD 1080p | 1920 | 1080 |
| 4K UHD | 3840 | 2160 |

Custom sizes are supported — any positive integer width and height.

---

## 4. Coordinate System

- **(0, 0)** is the **top-left** corner of the artboard.
- **X increases to the right**, **Y increases downward**.
- All coordinates are in **world space** (absolute pixels, not relative to a parent).
- Layers can extend beyond artboard boundaries.
- Rotation is in **radians**, clockwise, around the layer's center point.

---

## 5. Layer IDs

Every layer and comment has a unique string `id` in this format:

```
el_<timestamp_base36>_<counter_base36>
```

**Example:** `el_m1a2b3c_1`, `el_m1a2b3c_2`, `el_m1a2b3c_a`

- The timestamp prefix is a base-36 encoding of a millisecond timestamp.
- The counter suffix increments in base 36: `1`, `2`, ... `9`, `a`, `b`, ... `z`, `10`, `11`, ...
- All IDs within a document must be unique across layers, group children, and comments.

---

## 6. Layer Types

There are **11 layer types**: `rectangle`, `ellipse`, `line`, `arrow`, `star`, `polygon`, `triangle`, `text`, `image`, `path`, `group`.

### 6.1 Common Properties

Every layer object includes these properties:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `id` | string | — | Unique layer ID |
| `type` | string | — | One of the 11 layer types |
| `name` | string | Capitalized type | Display name |
| `x` | number | 0 | Left edge X position (world coords) |
| `y` | number | 0 | Top edge Y position (world coords) |
| `width` | number | 100 | Width in pixels |
| `height` | number | 100 | Height in pixels |
| `rotation` | number | 0 | Rotation in radians, clockwise, around center |
| `flipX` | boolean | false | Flip layer horizontally |
| `flipY` | boolean | false | Flip layer vertically |
| `opacity` | number | 1 | Overall opacity (0–1) |
| `visible` | boolean | true | Whether the layer is rendered |
| `locked` | boolean | false | Whether the layer is locked from editing |
| `aspectLocked` | boolean | false | Whether aspect ratio is locked on resize |
| `fill` | string | `"#cccccc"` | Fill color (hex string or `"transparent"`) |
| `fillEnabled` | boolean | true | Whether the fill is rendered |
| `fillOpacity` | number | 1 | Fill opacity (0–1) |
| `stroke` | string | `"#333333"` | Stroke color (hex string or `"transparent"`) |
| `strokeEnabled` | boolean | true | Whether the stroke is rendered |
| `strokeOpacity` | number | 1 | Stroke opacity (0–1) |
| `strokeWidth` | number | 1 | Stroke width in pixels |
| `strokeAlign` | string | `"center"` | Stroke alignment: `"center"`, `"inside"`, or `"outside"` |
| `strokeJoin` | string | `"miter"` | Stroke line join: `"miter"`, `"round"`, or `"bevel"` |
| `cornerRadius` | number | 0 | Corner rounding in pixels |

All properties must be present on every layer — there are no optional common properties.

### 6.2 Rectangle

The default shape. Used for buttons, cards, containers, input fields, backgrounds, etc.

- `cornerRadius` rounds all four corners equally, clamped to half the smaller dimension.
- For a pill/capsule shape, set `cornerRadius` to half the `height`.

Optional cover image properties:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `imageAssetId` | string \| null | null | ID of an asset in the `assets/` folder to display as a cover image |
| `imageName` | string \| null | null | Original filename of the cover image |
| `imageHidden` | boolean | false | Whether the cover image is temporarily hidden |

When `imageAssetId` is set, the image is rendered inside the shape using **object-fit: cover** (centered, cropped to fill, maintaining aspect ratio).

### 6.3 Ellipse

For circular or oval shapes — avatars, status indicators, radio buttons.

- For a perfect circle, set `width` equal to `height`.
- Use `aspectLocked: true` to maintain the ratio.
- Supports the same optional cover image properties as Rectangle (`imageAssetId`, `imageName`, `imageHidden`).

### 6.4 Line

Horizontal or angled lines — dividers, separators, underlines.

- `fill` must be `"transparent"`.
- The line draws from `(x, y)` to `(x + width, y + height)`.
- `height: 0` produces a horizontal line.
- `strokeWidth` is the line thickness.
- `cornerRadius > 0` enables rounded line caps.

### 6.5 Arrow

Same as line, but with an arrowhead at the end point.

- `fill` must be `"transparent"`.
- Arrowhead is drawn at `(x + width, y + height)`.
- Arrowhead size: `min(14, lineLength * 0.3)`.

### 6.6 Star

Multi-pointed star shape.

Additional properties:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `points` | integer | 5 | Number of outer points |

- Inner radius ratio is fixed at 0.4.
- Oriented with one point facing up.

### 6.7 Polygon

Regular polygon with configurable sides.

Additional properties:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `sides` | integer | 6 | Number of sides |

### 6.8 Triangle

Shortcut for a 3-sided polygon.

Additional properties:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `sides` | integer | 3 | Always `3` |

### 6.9 Text

Text labels, headings, body text, button labels.

Additional properties:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `text` | string | `"Text"` | The text content |
| `fontSize` | number | 16 | Font size in pixels |
| `fontFamily` | string | `"Roboto, sans-serif"` | Font family |
| `fontWeight` | string | `"normal"` | `"normal"`, `"bold"`, or `"100"`–`"900"` |
| `textAlign` | string | `"left"` | `"left"`, `"center"`, or `"right"` |

**Available font families:**
- `"Roboto, sans-serif"` (default)
- `"Poppins, sans-serif"`
- `"Merriweather, serif"`
- `"Caveat, cursive"`
- `"Dancing Script, cursive"`
- `"Shadows Into Light Two, cursive"`
- `"Roboto Mono, monospace"`
- `"BLOKK Neue, sans-serif"` (wireframe placeholder font)

Any CSS font-family string is accepted in the format; the above are the fonts bundled with the application.

**Rendering rules:**
- `fill` is the **text color** (not a background — default `"#000000"`).
- `stroke` must be `"transparent"` and `strokeWidth` must be `0`.
- Text wraps within the layer's `width`.
- Line height is `fontSize × 1.3`.
- Layer `height` should be approximately `(number of lines) × fontSize × 1.3`.

### 6.10 Path

Custom vector paths drawn with the Pen or Pencil tools — freehand sketches, bezier curves, custom shapes.

Additional properties:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `pathPoints` | array | `[]` | Ordered list of anchor point objects |
| `closed` | boolean | false | Whether the path forms a closed shape |

Each anchor point object:

| Property | Type | Description |
|----------|------|-------------|
| `x` | number | X position relative to the layer's origin |
| `y` | number | Y position relative to the layer's origin |
| `cp1x` | number | Incoming bezier control point X offset from anchor |
| `cp1y` | number | Incoming bezier control point Y offset from anchor |
| `cp2x` | number | Outgoing bezier control point X offset from anchor |
| `cp2y` | number | Outgoing bezier control point Y offset from anchor |

**Rendering rules:**
- Segments between anchor points are straight lines when both control point offsets are `(0, 0)`, otherwise cubic bezier curves.
- When `closed` is `true`, a closing segment connects the last point back to the first (with bezier handles if present), and the path is filled.
- When `closed` is `false`, `fill` defaults to `"transparent"` (stroke only).
- Line cap is `round`. The `strokeJoin` property defaults to `"round"` for paths (vs `"miter"` for other layer types).
- The layer's `x`, `y` define the origin — each point's `x`, `y` are relative to this origin.
- The layer's `width` and `height` are the bounding box of all anchor points.

### 6.11 Image

Displays an embedded image from the `assets/` folder.

Additional properties:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `assetId` | string \| null | null | ID of the asset in the `assets/` folder |

**Rendering rules:**
- `fill` and `stroke` are both `"transparent"`, `strokeWidth` is `0`.
- When `assetId` is `null`, renders as a light gray rectangle with an X through it (placeholder).
- Use `aspectLocked: true` to maintain image proportions.

### 6.12 Group

Contains other layers as children.

Additional properties:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `children` | array | `[]` | Ordered list of child layer objects |

**Rules:**
- `fill` and `stroke` are `"transparent"`, `strokeWidth` is `0`.
- The group's `x`, `y`, `width`, `height` must be the **bounding box** of its children:
  - `x` = min of all children's `x`
  - `y` = min of all children's `y`
  - `width` = max(`child.x + child.width`) − min(`child.x`)
  - `height` = max(`child.y + child.height`) − min(`child.y`)
- **Children use absolute world coordinates**, not positions relative to the group.
- Groups can be nested.

---

## 7. Layer Ordering

Layers in the `layers` array are rendered **bottom to top** — index 0 is drawn first (behind everything), and the last element is drawn on top.

---

## 8. Comments

Comments are stored separately from layers and appear as numbered pins on the canvas.

```json
{
  "id": "el_abc123_b",
  "x": 200,
  "y": 400,
  "text": "Revisit this button style",
  "resolved": false,
  "createdAt": 1711900000000
}
```

| Property | Type | Description |
|----------|------|-------------|
| `id` | string | Unique ID (same format as layers) |
| `x` | number | X position on the artboard |
| `y` | number | Y position on the artboard |
| `text` | string | Comment body |
| `resolved` | boolean | Whether the comment is resolved |
| `createdAt` | integer | Unix timestamp in milliseconds |

---

## 9. Assets

Image files are stored in the `assets/` folder with filenames in the format `<assetId>.<ext>` (e.g., `a1b2c3.png`).

Image layers reference their asset by `assetId`. The extension is determined by the original file's type.

The `assetManifest` field in `document.json` is reserved for future use and should be an empty array `[]`.

---

## 10. Thumbnail

On save, the application generates a `thumbnail.png` in the ZIP root. This is a downscaled PNG rendering of the artboard, fitting within **400×300 pixels** while preserving aspect ratio.

The thumbnail is optional — readers should not depend on it being present. It's intended for use by file browsers, galleries, or preview UIs.

---

## 11. Implementation Notes

### Reading `.npkd` Files

1. Open the file as a ZIP archive.
2. Parse `manifest.json` — verify `app === "Napukin"` and check `version`.
3. Parse `document.json` — read artboard dimensions, iterate layers.
4. Load assets from the `assets/` folder as needed (match by `assetId`).
5. Optionally read `thumbnail.png` for preview display.

### Writing `.npkd` Files

1. Create a ZIP archive with **STORE compression** (no deflation).
2. Add `manifest.json` with app metadata.
3. Add `document.json` with the full document model.
4. Add an `assets/` folder containing any embedded images.
5. Optionally render the artboard to a downscaled PNG and add as `thumbnail.png`.
6. Save with the `.npkd` extension.

### Generating `.npkd` from Code

Since the format is JSON inside an uncompressed ZIP, any language with a ZIP library can produce valid `.npkd` files. The minimum viable file contains:

- `manifest.json` with `app`, `version`, `name`, and `createdAt`
- `document.json` with `name`, `version`, `artboardWidth`, `artboardHeight`, `layers` (can be `[]`), `comments` (`[]`), and `assetManifest` (`[]`)
- An empty `assets/` folder

### Known Consumers

- **Napukin** (openNapkin) — the primary editor
