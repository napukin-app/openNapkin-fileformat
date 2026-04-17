# openNapkin File Format (`.npkd`)

The `.npkd` file format is the native document format for [Napukin](../README.md), a wireframing tool. This folder contains the full specification and resources for working with `.npkd` files.

## Overview

A `.npkd` file is a **plain, uncompressed ZIP archive** (STORE mode) with a structured set of JSON documents and optional binary assets inside. It's designed to be:

- **Simple** — human-readable JSON at its core, easy to inspect and generate
- **Portable** — a single self-contained file with all assets embedded
- **Tooling-friendly** — any language with a ZIP library can read or write `.npkd` files
- **AI-friendly** — LLMs can generate valid `document.json` from natural language descriptions of UI layouts

## What's Inside a `.npkd` File

```
MyDesign.npkd (ZIP archive, STORE/no compression)
├── manifest.json      ← App metadata, version, document name
├── document.json      ← Artboard dimensions, layer tree, comments
├── thumbnail.png      ← Low-res preview of the artboard (max 400×300px)
└── assets/            ← Embedded images (optional, can be empty)
    ├── <assetId>.png
    └── <assetId>.jpg
```

| File | Required | Description |
|------|----------|-------------|
| `manifest.json` | Yes | Identifies the file as a Napukin document with version and creation metadata |
| `document.json` | Yes | The full design: artboard size, layers, comments, and asset references |
| `thumbnail.png` | No | Preview image, generated on save, max 400×300px |
| `assets/` | Yes | Folder for embedded image files, referenced by ID from image layers |

## Documents in This Folder

| Document | Description |
|----------|-------------|
| [npkd-spec.md](npkd-spec.md) | Full technical specification — file structure, JSON schemas, layer types, coordinate system, rendering rules |

## Quick Start

### Reading a `.npkd` File

```js
import JSZip from 'jszip';

const zip = await JSZip.loadAsync(file);
const manifest = JSON.parse(await zip.file('manifest.json').async('string'));
const document = JSON.parse(await zip.file('document.json').async('string'));

console.log(manifest.name);              // "My Login Screen"
console.log(document.artboardWidth);     // 393
console.log(document.layers.length);     // 15
```

### Writing a `.npkd` File

```js
import JSZip from 'jszip';

const zip = new JSZip();
zip.file('manifest.json', JSON.stringify({
  app: 'Napukin',
  version: 1,
  name: 'My Design',
  createdAt: Date.now(),
}));
zip.file('document.json', JSON.stringify({
  name: 'My Design',
  version: 1,
  artboardWidth: 393,
  artboardHeight: 852,
  layers: [/* ... */],
  comments: [],
  assetManifest: [],
}));
zip.folder('assets');

const blob = await zip.generateAsync({ type: 'blob', compression: 'STORE' });
// Save blob as .npkd
```

### Creating One from the Command Line

```bash
# Given manifest.json, document.json, and an assets/ folder:
zip -0 MyDesign.npkd manifest.json document.json assets/
```

## Use Cases

- **Code-generated wireframes** — build `.npkd` files programmatically from design systems, component libraries, or data models
- **AI wireframe generation** — give an LLM the [spec](npkd-spec.md) and it can produce valid `.npkd` documents from natural language
- **Design tooling integration** — read `.npkd` files to extract layer data, generate code, or convert to other formats
- **Version control** — since the ZIP uses STORE mode and the contents are JSON, diffs are meaningful when extracted
