# AC Map Editor

A production-grade tile map editor for **Armor Critical**, the classic top-down multiplayer arena shooter. One TypeScript codebase ships as a Windows desktop app, a Linux desktop app, and a fully in-browser web editor — reading and writing the game's binary map format with byte-for-byte fidelity to the original 1990s-era tooling it replaces.

Built by [ArcBound Interactive](https://arcboundinteractive.com).

**Stats:** 1,005 commits · TypeScript/React (~30,000 lines across app, rendering core, and desktop shell) · Electron + Vite · 90+ development phases completed

> This repository contains documentation only. The source code is private; this overview describes the architecture and engineering of the project.

---

## Novel Systems & Methods

These are the systems in the editor that go beyond standard CRUD-editor fare — each one solves a problem that off-the-shelf libraries do not.

### 1. Content-Aware Map Transforms
Mirroring or rotating a selection in a tile map normally just moves tiles — which shreds anything with structure: walls end up visually disconnected, multi-tile objects (flags, spawn pads, warp gates) get scrambled, and directional objects (conveyors, bunkers, bridges) point the wrong way. The editor's transform engine understands the *semantics* of every tile it moves. It reverse-maps each tile back to its logical object, applies the geometric transform to the object's connection state or orientation (wall connection bitmasks are rotated/reflected as 4-bit neighbor masks; 3×3 object footprints are re-assembled via position remap tables), then re-emits the correct tile variants. The result: rotate a whole base 90 degrees and every wall junction, bunker opening, and conveyor direction is still correct.

### 2. Incremental Four-Layer Canvas Renderer
The map is a fixed 256×256 grid — 65,536 tiles, a 4096×4096 pixel surface. Rather than re-rendering per frame, the engine keeps a full-resolution off-screen buffer and layers the display:

- **Map layer** — the off-screen buffer is rendered once, then *patched incrementally*: an edit repaints only the changed tiles; a pan or zoom is a single `drawImage` blit.
- **Grid layer** — a cached `CanvasPattern`, rebuilt only when zoom or grid settings change.
- **UI layer** — cursor highlight, selection rectangles, and live tool previews, redrawn via a requestAnimationFrame-debounced dirty flag.
- **Text layer** — DPI-scaled canvas for crisp measurement labels.

Animated tiles are patched per-frame *only within the visible viewport* (dirty-rect updates). Measured results: <1 ms per-tile patch, ~1 ms viewport blit, ~8 ms input-to-paint latency — smooth 60 fps editing even during flood fills and drag-paints, using plain Canvas2D with no WebGL complexity.

### 3. Gesture State Machine for Unified Input
To make the same editor work with a mouse, a finger, and a stylus, all input flows through a `PointerManager` that implements an explicit gesture state machine (`IDLE → SINGLE_DOWN → TOOL_ACTIVE / NAVIGATING`). A short grace window disambiguates "start painting" from "start pinching"; a second finger arriving mid-stroke cancels the in-progress tool action and converts cleanly to pan/zoom; two-finger tap is undo, three-finger tap is redo; long-press invokes the tile picker; detecting a pen enables palm rejection (touch ignored for tool actions, still valid for navigation). The consuming canvas component receives clean, already-disambiguated callbacks — it never sees a raw pointer event.

### 4. Faithful Wall Auto-Connection Engine
Armor Critical maps use 15 wall families, each with 16 tile variants selected by which of the four neighbors are also walls. The editor re-implements the original editor's neighbor-bitmask connection algorithm so that freehand wall painting, wall rectangles, and wall lines all self-connect — and stay connected when neighboring walls are erased or transformed. This behavioral fidelity matters: maps produced here are indistinguishable from ones made with the community's legacy tooling, which kept two decades of existing maps editable.

### 5. Smart Crop via Flood-Fill Cluster Detection
For exporting map overview images, the editor auto-detects the actual play area: a BFS flood-fill segments the map into connected content clusters, keeps the largest cluster plus any cluster within a distance threshold, and discards distant outliers (e.g. off-map holding pens). The exported overview is tightly framed around the real playfield with zero manual cropping.

### 6. One Core, Three Platforms
All editor logic — parsing, tile encoding, wall/object systems, rendering, state — lives in a platform-agnostic core with **zero** Electron or browser-API imports. Thin adapters supply platform capability behind a single `FileService` interface: the Electron adapter uses native dialogs, IPC, and Node's zlib; the web adapter uses the File System Access API (with graceful fallbacks) and a pure-JS zlib. Entry points inject the right adapter through React context. The web build is not a port — it is the same application with a different ~500-line adapter.

### 7. Game-Object Placement System
Beyond tiles, the editor places *game objects*: team flags and poles, spawn pads, switches, five styles of warp gates with encoded routing, turrets, directional bunkers (4 styles × 4 directions), bridges, conveyors, and holding pens — each a multi-tile stamp with its own placement rules, some carrying encoded metadata inside reserved tile bits. Placement logic was matched against the original editor's behavior so objects function correctly in the live game.

### Also notable
- **Multi-document MDI workspace** — up to 8 maps open at once, each with its own viewport, selection, and 50-deep snapshot undo stack (with engine-level drag batching so a paint stroke is one undo step).
- **Built-in tileset and sprite pixel editors** — pencil/fill/eyedropper tools, rotate/flip, copy/paste, and a working-copy Apply/Revert flow so the source art is never touched until committed.
- **Ship sticker overlay** — stamp ship sprites (4 teams × 9 facings) onto the map as a non-destructive overlay layer for planning and presentation.
- **30+ bundled community tileset patches**, selectable at runtime and swappable per-map.

---

## What It Does

- **Full map format support** — reads and writes all three generations of the binary `.map` format (raw, legacy, and current zlib-compressed), validated against the original game and community maps.
- **22 editing tools** — pencil (single and multi-tile stamp), flood fill, line, rectangle, select/move, copy/cut/paste with floating preview, three wall tools, ten game-object tools, a four-mode measuring ruler (line, rectangle, radius, waypoint path), and content-aware mirror/rotate.
- **Editor conveniences** — cursor-anchored wheel zoom (0.25×–4×), minimap, tabbed panels, tile animation preview, map settings editor covering the game's full settings block, overview image export with smart crop, theme system.
- **Distribution** — Windows installer and Linux `.deb` desktop builds with auto-update, plus a hosted browser version usable on any OS (including macOS) with no install.

## Status

Actively developed and in use by the Armor Critical map-making community. The desktop editions are the primary distribution; the web edition tracks desktop feature parity on desktop browsers, with a touch/tablet input layer (the gesture state machine above) built to extend it to mobile devices.

## Heritage

The editor is a modern successor to SEDIT, the community's original C++ map editor (by Wayne A. Witzel III, Jon-Pierre Gentile, and David Parton). Core behaviors — wall connection, object placement, format handling — were carefully matched so the two decades of existing community maps remain first-class citizens.

## Documentation

- [ARCHITECTURE.md](ARCHITECTURE.md) — layers, data flow, rendering pipeline, and the desktop/web split, with diagrams
- [SETUP.md](SETUP.md) — toolchain and build overview
