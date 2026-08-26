# Setup & Toolchain

This page describes the development environment and build pipeline at a general level. (The repository is private; exact configuration files are not reproduced here.)

## Toolchain

| Component | Choice | Role |
|-----------|--------|------|
| Language | TypeScript 5 (strict mode) | All application and desktop-shell code |
| UI | React 18 | Component layer |
| State | Zustand | Sliced global store (documents / global / window) |
| Build | Vite 6 | Dev server with fast refresh; production bundling |
| Desktop shell | Electron 34 | Windows and Linux desktop editions |
| Packaging | electron-builder | NSIS installer (Windows), `.deb` (Linux), auto-update feed |
| Compression | Node `zlib` (desktop) / pako (web) | Map format compression behind a shared interface |
| Layout/UX libs | react-resizable-panels, react-rnd, react-icons | Panel resizing, MDI windows, iconography |

Runtime dependencies are deliberately minimal — no HTTP clients, no database, no analytics. The editor is fully local-first.

## Prerequisites

- Node.js 18 or newer (Node 20+ recommended)
- npm

That is the entire requirement set; no native build tools or global CLIs are needed.

## Development Workflow

```bash
npm install        # install dependencies
npm run dev        # Vite dev server + Electron with hot reload
npm run typecheck  # strict TypeScript check, no emit
```

The dev loop runs Vite's dev server and launches Electron against it, so renderer changes hot-reload instantly while the Electron main process restarts on change.

## Build Targets

Three production targets build from the same source tree:

| Target | Command (conceptually) | Output |
|--------|------------------------|--------|
| Windows desktop | Vite build + electron-builder (win) | One-click NSIS installer |
| Linux desktop | Vite build + electron-builder (linux) | `.deb` package with desktop entry |
| Web | Vite build with a web-specific config | Static SPA — deployable to any static host |

Notes on the split:

- **Two Vite configs, two HTML shells.** The desktop config wires the Electron main-process and preload builds via Vite's Electron plugins; the web config builds a plain SPA from a separate entry point that injects the browser file-service adapter.
- **Bundled assets.** Tilesets and 30+ community tileset patches ship inside the app (as extra resources on desktop, as static assets on web), so the editor is fully usable offline with no external fetches.
- **Auto-update.** Desktop builds check a release feed for updates; the web edition simply redeploys as static files.
- **Linux specifics.** The Linux package registers a desktop entry under the Graphics category; a small post-pack step adjusts Chromium sandbox handling for distributions that require it.

## Type Discipline

The TypeScript configuration enforces strict mode, no unused locals/parameters, and no switch fallthrough, with path aliases for the core (`@core/*`) and component (`@components/*`) layers. The core layer compiles with zero platform type dependencies — importing an Electron or DOM-file-API type into `src/core/` is a build error by convention and review, which is what keeps the web/desktop split honest.

## Verifying a Build

1. `npm run typecheck` passes clean.
2. Desktop: launch the packaged app, open a community map, edit, save, and diff-verify a round-trip (open → save with no edits produces an identical file).
3. Web: serve the web build output from any static file server and repeat the round-trip using the browser's file picker.

The round-trip check is the project's core invariant: the editor must never alter map bytes it did not intend to.
