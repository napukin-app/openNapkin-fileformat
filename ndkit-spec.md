# `.ndkit` File Format Specification

**Version:** 1
**Application:** Napukin (openNapkin)

A `.ndkit` file is an **openNapkin Design Kit** — a portable package of pre-made,
reusable UI components (a design system) that users can browse in the editor's
Assets tab and insert into their documents.

---

## 1. File Structure

A `.ndkit` file is a **ZIP archive using STORE compression** (no deflation),
mirroring the `.npkd` document format. It contains:

```
├── manifest.json       (required)
├── thumbnail.png       (optional — kit cover image)
├── components/         (required)
│   ├── <componentId>.json
│   └── ...
└── assets/             (optional)
    ├── <assetId>.png
    └── ...
```

The MIME type is `application/zip`. The file extension is `.ndkit`.

---

## 2. manifest.json

```json
{
  "format": "ndkit",
  "formatVersion": 1,
  "id": "napkin-essentials",
  "name": "Napukin Essentials",
  "description": "Core UI building blocks: buttons, inputs, toggles and cards.",
  "kitVersion": "1.0.0",
  "author": "Napukin",
  "components": [
    { "id": "button-primary", "name": "Button / Primary", "category": "Buttons" }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `format` | string | yes | Always `"ndkit"` |
| `formatVersion` | integer | yes | Format version, currently `1` |
| `id` | string | yes | Globally unique kit identifier (lowercase, `a-z0-9-`) |
| `name` | string | yes | Human-readable kit name |
| `description` | string | no | Short description shown in the Assets tab |
| `kitVersion` | string | yes | Kit content version (semver recommended) |
| `author` | string | no | Kit author |
| `components` | array | yes | Directory of components in the kit |

Each entry in `components` must have a matching `components/<id>.json` file.
`category` is optional and used for grouping/search in the Assets tab.

---

## 3. components/&lt;componentId&gt;.json

Each component is stored as one JSON file:

```json
{
  "id": "button-primary",
  "name": "Button / Primary",
  "category": "Buttons",
  "defaultState": "default",
  "states": {
    "default": { "...": "group layer tree" },
    "hover":   { "...": "group layer tree" },
    "pressed": { "...": "group layer tree" }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | yes | Unique within the kit, must match the filename |
| `name` | string | yes | Display name (e.g. `"Button / Primary"`) |
| `category` | string | no | Grouping label (e.g. `"Buttons"`) |
| `defaultState` | string | yes | Key into `states` — the state inserted by default |
| `states` | object | yes | Map of state name → layer tree (at least one state) |

### 3.1 State layer trees

Each state is a **complete layer tree** in the `.npkd` layer format (see
`npkd-spec.md` §6). The root layer should be a `group` whose `name` is the
component name, containing the component's parts as children (e.g. a
background `rectangle` and a label `text` layer). This keeps inserted
components fully editable with native tools.

Rules:

- The root layer's `x`/`y` must be `0, 0` (the tree is **normalized to the
  origin**). Children use absolute coordinates relative to that same origin,
  matching the `.npkd` convention that group children are in world space.
- The root's `width`/`height` must be the bounding box of its children.
- Layer `id`s only need to be unique within the component — the editor
  regenerates all IDs on insertion.
- Layers may omit common properties; readers (and the kit builder CLI)
  normalize missing properties to the `.npkd` defaults.
- Image layers reference kit assets by bare `assetId` (`assetId` /
  `imageAssetId`), resolved against the kit's `assets/` folder.

### 3.2 Multi-state components

Components with interaction states (default / hover / pressed / focus / …)
store each state as its own complete tree under `states`. The editor:

1. Always inserts the `defaultState` tree.
2. Tags the inserted root group with component metadata
   (`{ kitId, kitName, kitVersion, componentId, state }`).
3. Lets the user switch states later from the properties panel — the group's
   children are swapped for the chosen state's tree (position preserved, text
   edits carried over by matching layer names).

State names are free-form, but `default`, `hover`, `pressed`, `focus` and
`disabled` are conventional.

---

## 4. Assets

Image files used by components live in `assets/` with filenames in the
format `<assetId>.<ext>` — identical to the `.npkd` assets folder.

When a component is inserted, the editor copies the assets it references into
the document under namespaced IDs (`kit_<kitId>_<assetId>`) so repeated
insertions deduplicate and kit assets never collide with document assets.

---

## 5. Thumbnail

`thumbnail.png` is an optional cover image for the kit shown in the Assets
tab (recommended around 320×200). Individual **component previews are
rendered live by the editor** from the layer trees — they are not stored in
the package.

---

## 6. Hosting kits

The web app loads kits from the `kits/` folder served next to the app
(`public/kits/` in the repository):

```
public/kits/
├── index.json          — ["napkin-essentials.ndkit", "my-kit.ndkit"]
└── napkin-essentials.ndkit
```

`index.json` is a JSON array of `.ndkit` filenames; static hosts cannot list
directories, so the index is required. Self-hosters choose which kits their
instance offers by editing this folder.

Users can also import a local `.ndkit` for the current session via the
**Import kit** button in the Assets tab.

---

## 7. Kits embedded in `.npkd` documents

When a document uses kit components, saving embeds the used component
definitions (and only the assets they reference) into the `.npkd` under a
`kits/` folder — see `npkd-spec.md` §12. This keeps documents portable:
state switching keeps working even on hosts that don't serve the kit.

---

## 8. Authoring kits

The repository ships a builder CLI:

```
npm run build-kit -- <kit-source-dir> [output.ndkit]
```

A kit source directory is the unpacked layout from §1. The builder validates
the manifest and components (known layer types, resolvable asset references,
matching IDs), normalizes missing layer properties to `.npkd` defaults, and
packages a STORE-compressed `.ndkit`.

Since the format is JSON inside an uncompressed ZIP, any language with a ZIP
library can also produce valid `.ndkit` files.
