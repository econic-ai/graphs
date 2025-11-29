# Econic Graphs

**A framework for building interactive 3D graph visualisations in the browser.**

This monorepo contains the architecture, documentation, and source packages for a multi-layered framework that enables developers to render dynamic, hierarchical graphs in 3D space using WebGPU — with the flexibility to overlay their own HTML/SVG components on node positions.

## What's This About?

Ever wanted to build a beautiful 3D network visualisation where:
- Nodes can be **clustered, collapsed, and expanded** with smooth animations?
- You control the **visual design** with your own HTML/CSS/SVG?
- Performance scales to **thousands of nodes** via GPU rendering?

That's what we're building here.

## Architecture

The framework is designed as composable layers, each solving a distinct problem:

```
┌─────────────────────────────────────────────────────────────┐
│  @econic/graph-ui (Future)                                  │
│  Framework bindings (React/Vue/Svelte) for declarative APIs │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│  @econic/graph-morph                                        │
│  Meta-graph management, projections, animated transitions   │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│  @econic/graph-3d                                           │
│  Visible nodes, HTML overlays, user interaction, buffers    │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│  @econic/graph-core (Rust/WASM)                             │
│  High-performance WebGPU rendering and projection           │
└─────────────────────────────────────────────────────────────┘
```

## Packages

| Package | Language | Description |
|---------|----------|-------------|
| [`graph-core`](./graph-core/) | Rust/WASM | WebGPU rendering engine — projects 3D positions to screen coordinates |
| [`graph-3d`](./graph-3d/) | TypeScript | Orchestrates buffers, manages overlays, handles user interaction |
| [`graph-morph`](./graph-morph/) | TypeScript | Hierarchical meta-graph with expand/collapse transitions and layouts |
| [`graph-ui`](./graph-ui/) | TypeScript | *(Future)* React, Vue, Svelte bindings for declarative component APIs |

Each package is published separately to npm under the `@econic/` scope.

## Key Concepts

### The Meta-Graph

Your graph data lives in a hierarchical **meta-graph**. Nodes can be grouped, and groups can be collapsed into summary nodes or expanded to reveal children. The visible graph at any moment is a **projection** of this structure.

### 2D Overlays on 3D Positions

The framework computes 3D spatial relationships on the GPU, but you provide the visuals. Attach any HTML element to a node — cards, badges, charts, whatever you like — and the framework positions them in screen space as the camera moves.

### Zero-Copy Buffer Communication

Position data flows between JavaScript and WASM via shared `Float32Array` buffers. No serialisation overhead. The GPU renders, writes screen positions back to buffers, and your overlays update via CSS transforms.

## Getting Started

> 🚧 **Work in Progress** — We're building this in the open. Check back for setup instructions.

## Documentation

Detailed architecture, API specifications, and design decisions live in [`/docs`](./docs/).

- [Project Scope & Architecture](./docs/Project-scope.md) — The full blueprint

## Development

This repo is organised as a **multi-repo workspace** — each package is its own git repository:

```bash
# Clone the workspace
git clone https://github.com/econic/graphs.git

# Each package has its own origin
cd graph-core && git remote -v
cd graph-3d && git remote -v
# etc.
```

### Prerequisites

- **Rust** (for `graph-core`) — [rustup.rs](https://rustup.rs)
- **wasm-pack** — `cargo install wasm-pack`
- **Node.js 20+** and **pnpm** — `npm install -g pnpm`

### Building

```bash
# Build the WASM core
cd graph-core
wasm-pack build --target web

# Build TypeScript packages
cd ../graph-3d && pnpm install && pnpm build
cd ../graph-morph && pnpm install && pnpm build
```

## License

MIT — see individual packages for details.

---

Built with 🧠 by [Econic](https://econic.ai)

