# `.npkd` File Format Specification

**Manifest version:** 1  
**Document version:** 2 (multi-page; version 1 files are still readable — see §3.3)  
**Application:** Napukin (openNapkin)

---

## 1. File Structure

A `.npkd` file is a **ZIP archive using STORE compression** (no deflation). It contains:

```
├── manifest.json       (required)
├── document.json       (required)
├── thumbnail.png       (optional)
├── assets/             (required, may be empty)
│   ├── <id>.png
│   ├── <id>.jpg
│   └── ...
└── kits/               (optional — embedded UI kits, see §12)
    └── <kitId>/
        ├── manifest.json
        ├── components/<componentId>.json
        └── assets/<assetId>.<ext>
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

A document is **multi-page** (format version 2). Each page owns its own artboard
size/fill, layers, and comments; assets are shared across the whole document.

```json
{
  "name": "My Design",
  "version": 2,
  "canvasBackground": "#2c2c2c",
  "activePageId": "el_m1a2b3c_1",
  "pages": [
    {
      "id": "el_m1a2b3c_1",
      "name": "Page 1",
      "artboardWidth": 393,
      "artboardHeight": 852,
      "artboardFill": true,
      "artboardFillColor": "#ffffff",
      "layers": [],
      "comments": []
    }
  ],
  "artboardWidth": 393,
  "artboardHeight": 852,
  "artboardFill": true,
  "artboardFillColor": "#ffffff",
  "usedKits": [],
  "assetManifest": []
}
```

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Document name (matches manifest) |
| `version` | integer | Document format version, currently `2` (multi-page) |
| `canvasBackground` | string | Color of the canvas area outside the artboard, as hex string (default `"#2c2c2c"`) |
| `activePageId` | string | `id` of the page that was active when the file was saved |
| `pages` | array | Ordered list of page objects (see §3.1) |
| `artboardWidth` | number | Active page's artboard width, mirrored at the top level for backward compatibility (see §3.2) |
| `artboardHeight` | number | Active page's artboard height, mirrored at the top level |
| `artboardFill` | boolean | Active page's artboard fill flag, mirrored at the top level |
| `artboardFillColor` | string | Active page's artboard background color, mirrored at the top level |
| `usedKits` | array | Summary of UI kits referenced by component instances across all pages: `[{ id, name, kitVersion }]` (derived from layers; optional) |
| `assetManifest` | array | Summary of embedded assets: `[{ id, name, type }]` (derived from referenced assets; optional) |

### 3.1 Pages

Each entry in the `pages` array is a page object:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | string | — | Unique page ID (same format as layer IDs, see §5) |
| `name` | string | `"Page N"` | Human-readable page name |
| `artboardWidth` | number | 393 | Page artboard width in pixels |
| `artboardHeight` | number | 852 | Page artboard height in pixels |
| `artboardFill` | boolean | `true` | Whether the page artboard has a background fill |
| `artboardFillColor` | string | `"#ffffff"` | Page artboard background color as hex string |
| `layers` | array | `[]` | Ordered list of layer objects for this page (bottom to top) |
| `comments` | array | `[]` | List of comment objects for this page |

Rules:

- A document always has **at least one page**; the last page cannot be deleted.
- `activePageId` should match one of the pages' `id`s. If it's missing or
  doesn't match, readers fall back to the first page.
- Layer IDs must be unique **across the whole document**, not just within a page.

### 3.2 Top-level artboard mirror

For backward compatibility with tooling that expects a single artboard, the
**active page's** `artboardWidth`, `artboardHeight`, `artboardFill`, and
`artboardFillColor` are also written at the top level of `document.json`. These
top-level fields are a read-only mirror — the authoritative values live on each
page object. Writers should keep them in sync with the active page; readers that
support pages should ignore them in favor of the per-page values.

### 3.3 Versioning & backward compatibility

- **Version 2** (current): multi-page documents with a `pages` array and
  `activePageId`. Layers and comments live inside each page object.
- **Version 1** (legacy): a single artboard with top-level `layers` and
  `comments` arrays and no `pages`. Readers should detect a missing/empty
  `pages` array and migrate the document into a single page named `"Page 1"`,
  using the top-level `artboardWidth`/`artboardHeight`/`artboardFill`/
  `artboardFillColor` and `layers`/`comments`. Napukin performs this migration
  automatically on open.

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
- `cornerRadii` (optional object `{ tl, tr, br, bl }`) overrides `cornerRadius` per corner, enabling independent corner radii. When all four values are equal, writers should omit `cornerRadii` and use the uniform `cornerRadius` instead. Any corner missing from the object falls back to `cornerRadius`.

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
| `textAlign` | string | `"left"` | `"left"`, `"center"`, or `"right"` (horizontal alignment) |
| `lineHeight` | number | `round(fontSize × 1.3)` | Line height in **pixels** (same unit as `fontSize`) |
| `verticalAlign` | string | `"middle"` | Vertical alignment within the layer box: `"top"`, `"middle"`, or `"bottom"` |

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
- Each line occupies `lineHeight` pixels (a CSS-style line box). When
  `lineHeight` is absent, fall back to `fontSize × 1.3`.
- `verticalAlign` positions the block of wrapped lines within the layer's
  `height`: `"top"` aligns to the top edge, `"middle"` centers it, `"bottom"`
  aligns to the bottom edge.
- Layer `height` should be approximately `(number of lines) × lineHeight`.

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

**Component instances:**

A group inserted from a UI kit (see `ndkit-spec.md`) carries an optional
`component` property:

```json
"component": {
  "kitId": "napkin-essentials",
  "kitName": "Napukin Essentials",
  "kitVersion": "1.0.0",
  "componentId": "button-primary",
  "state": "default"
}
```

Readers that don't understand kits can ignore this property — the group
renders like any other group. The editor uses it to show the component's
state switcher and to track which kits a document uses.

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
3. Parse `document.json`. If `pages` is present (version 2), iterate each page's
   artboard, layers, and comments. If `pages` is absent (version 1), treat the
   top-level artboard fields, `layers`, and `comments` as a single page (see §3.3).
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
- `document.json` with `name`, `version: 2`, `activePageId`, and a `pages` array
  containing at least one page (`id`, `name`, `artboardWidth`, `artboardHeight`,
  `layers` can be `[]`, `comments` can be `[]`). The top-level artboard mirror
  (§3.2) and `assetManifest` (`[]`) are recommended for compatibility.
- An empty `assets/` folder

### Known Consumers

- **Napukin** (openNapkin) — the primary editor
- **Gemini Gem** — an AI prompt that generates `.npkd` JSON from natural language (see [gemini-gem-npkd.md](gemini-gem-npkd.md))

---

## 12. Embedded UI Kits (`kits/`)

When a document contains component instances from UI kits (`.ndkit` files —
see [ndkit-spec.md](ndkit-spec.md)), the editor embeds the **used** component
definitions into the document on save, so the file stays portable: opening it
on a host that doesn't serve the kit still shows the kit under "In use in
this file" in the Assets tab and keeps state switching working.

```
kits/
└── <kitId>/
    ├── manifest.json               — ndkit manifest, components[] limited to used ones
    ├── components/
    │   └── <componentId>.json      — only components used by this document
    └── assets/
        └── <namespacedAssetId>.<ext>
```

Rules:

- Only components actually referenced by a layer's `component` metadata are
  embedded, along with only the assets those components' states reference.
- Embedded kit assets are stored under their **namespaced IDs**
  (`kit_<kitId>_<assetId>`) — the same IDs the component trees reference and
  the same IDs used in the document's own `assets/` folder for inserted
  instances.
- Component state trees inside an embedded kit already use namespaced asset
  references (unlike a standalone `.ndkit`, which uses bare asset IDs).
- The `kits/` folder is optional. Readers that don't support kits can ignore
  it entirely; component instances are plain groups.
