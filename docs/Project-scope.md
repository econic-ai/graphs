# Evolving the UI — A 3D User Interface Framework

*Architectural Blueprint for AI-Assisted Development*

---

## Executive Summary

This document defines the architecture for an industry-first framework enabling developers to build interactive 3D graph visualisations in the browser. The framework combines high-performance WebGPU rendering (via Rust/WASM) with a flexible JavaScript API that allows developers to overlay their own HTML/SVG components on 3D node positions.

The framework is designed as a set of composable packages, each solving a distinct problem, publishable as open-source projects.

---

## What We Are Building

A multi-layered framework for displaying and animating dynamic graphs in 3D space within the browser. The core innovation is the separation of concerns:

1. **3D spatial relationships** are computed on the GPU for performance
2. **Visual representation** is delegated to the developer via HTML/SVG overlays
3. **Graph structure and transformations** are managed by a meta-graph that supports hierarchical collapse/expand operations

The visible graph at any moment is a **projection** of an underlying hierarchical meta-graph. Nodes can be grouped, collapsed into summary nodes, and expanded — with smooth animated transitions between states.

### Target Use Cases

**Use Case 1: Animated Website Visualisation**
A JavaScript-driven animation of a graph on a marketing website. As the user interacts with the page (clicks, scrolls, selects options), the network transforms to illustrate different concepts. The transformations are scripted — the JS layer drives all changes.

**Use Case 2: Interactive Graph Editor**
A tool where users directly manipulate the graph — moving nodes, adding/removing connections, collapsing clusters, exploring hierarchies. The user's interactions drive the transformations.

Both use cases require the same underlying capabilities; they differ only in what triggers the transformations (script vs user input).

---

## Hard Requirements

### Coordinate System
- **Normalised Device Coordinates (NDC):** The framework operates in NDC space where `-1 to 1` on each axis represents the visible viewport. Positions outside this range are valid but may be clipped.
- The camera transforms world positions to NDC; the projection then maps to screen pixels.

### Performance Architecture
- **Dirty flag propagation:** GPU/CPU work only occurs when state has changed. Camera movements, node position updates, and structural changes set dirty flags. Unchanged frames skip rendering entirely.
- **Shared buffer communication:** Position data flows between JS and WASM via shared `Float32Array` buffers. No serialisation overhead on the hot path.
- **Screen position output:** After rendering, WASM writes projected 2D screen coordinates back to a buffer. JS reads these to position HTML overlays without additional computation.

### Rendering Model
- **GPU rendering:** Nodes are rendered as simple spheres (development) with the expectation that production use will overlay custom SVG/HTML elements on node positions.
- **2D overlay skinning:** The framework provides 3D spatial positioning; developers provide visual representation via HTML/SVG elements that track node screen positions.
- **Edge rendering:** Simple lines without depth. Complex edge visuals are implemented as 2D overlays in the JS layer.
- **Text rendering:** Handled entirely via HTML overlays, not GPU text rendering.

### Overlay System
- **Developer-provided container:** Overlays are appended to a container element specified by the developer, not managed internally.
- **Depth information:** Screen position output includes depth values. Developers can use this for z-index sorting, but the framework does not enforce a particular strategy.
- **No built-in hit testing:** If interactivity is required on nodes, developers attach click handlers to their overlay elements. The framework does not provide GPU-based picking.

### Graph Structure
- **Hierarchical meta-graph:** A tree structure (DAG in phase 2) overlays the visible graph, capturing parent-child containment relationships.
- **Projection model:** The visible graph is derived by projecting the meta-graph based on which group nodes are expanded or collapsed.
- **Summary edges:** Edges defined at group level are visual summaries shown only when endpoints are collapsed. Leaf-level edges are the source of truth.
- **Dynamic structure:** The meta-graph can be modified at runtime — nodes added, removed, or reparented.

### Animation and Transitions
- **Expand/collapse animations:** When a group node expands, its children animate outward from the group's position to their final positions. When collapsing, children converge to the group position before being replaced.
- **Position sources:** Node positions can be explicit (developer-specified), computed (via layout algorithms), or derived (centroid of children for group nodes).
- **Keyed animations:** Transitions interpolate between start and end states over a specified duration with configurable easing.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 4: @econic/graph-ui (Future)                             │
│  Framework bindings (React/Vue/Svelte), declarative components  │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│  Layer 3: @econic/graph-morph                                   │
│  Meta-graph, projection, transitions, layouts                   │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│  Layer 2: @econic/graph-3d                                      │
│  Visible nodes, overlays, events, buffer orchestration          │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: @econic/graph-core (Rust/WASM)                        │
│  WebGPU rendering, projection, culling                          │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
Meta-Graph (Layer 3)
    │
    │ Projection based on expansion state
    ▼
Visible Nodes + Positions (Layer 3 → Layer 2)
    │
    │ Written to shared Float32Array buffers
    ▼
World Positions Buffer (Layer 2 → Layer 1)
    │
    │ GPU rendering + view-projection transform
    ▼
Screen Positions Buffer (Layer 1 → Layer 2)
    │
    │ Read by JS to update overlay transforms
    ▼
HTML Overlays positioned via CSS transform
```

---

## Layer 1: @econic/graph-core (Rust/WASM)

### Responsibility

Raw rendering and GPU-side computations. This layer knows nothing about graphs — it renders points in space and projects them to screen coordinates.

### Inputs
- World positions (`Float32Array`, JS writes)
- Node scales (`Float32Array`, JS writes)
- Node visibility flags (`Uint8Array`, JS writes)
- Camera state (distance, pan, rotation)

### Outputs
- Screen positions (`Float32Array`, WASM writes, JS reads)
- Depth values (`Float32Array`, WASM writes, JS reads)
- Dirty flag signalling when outputs have been updated

### Public Interface

```rust
// lib.rs - WASM-exposed interface

use wasm_bindgen::prelude::*;
use js_sys::{Float32Array, Uint8Array};

#[wasm_bindgen]
pub struct GraphCore {
    // Internal renderer state
    surface: wgpu::Surface<'static>,
    device: wgpu::Device,
    queue: wgpu::Queue,
    // ... rendering pipeline, buffers, etc.
    
    // Shared buffer references (views into JS-owned memory)
    world_positions: Option<Float32Array>,
    scales: Option<Float32Array>,
    visibility: Option<Uint8Array>,
    screen_positions: Option<Float32Array>,
    depths: Option<Float32Array>,
    
    // State
    node_count: u32,
    camera: Camera,
    dirty: bool,
}

#[wasm_bindgen]
impl GraphCore {
    /// Create a new renderer attached to the given canvas
    #[wasm_bindgen(constructor)]
    pub async fn new(canvas: web_sys::HtmlCanvasElement, options: JsValue) -> Result<GraphCore, JsValue>;
    
    /// Set the maximum number of nodes (allocates GPU buffers)
    #[wasm_bindgen]
    pub fn set_capacity(&mut self, max_nodes: u32);
    
    /// Register shared buffers for zero-copy data exchange
    /// These buffers are owned by JS; Rust reads/writes directly
    #[wasm_bindgen]
    pub fn set_buffers(
        &mut self,
        world_positions: Float32Array,    // Input: [x, y, z, x, y, z, ...] per node
        scales: Float32Array,              // Input: [scale, scale, ...] per node
        visibility: Uint8Array,            // Input: [0|1, 0|1, ...] per node
        screen_positions: Float32Array,    // Output: [x, y, x, y, ...] screen coords
        depths: Float32Array,              // Output: [depth, depth, ...] per node
    );
    
    /// Set the number of active nodes (must be <= capacity)
    #[wasm_bindgen]
    pub fn set_node_count(&mut self, count: u32);
    
    /// Camera controls
    #[wasm_bindgen]
    pub fn set_camera(&mut self, distance: f32, pan_x: f32, pan_y: f32, rotation_x: f32, rotation_y: f32);
    
    #[wasm_bindgen]
    pub fn zoom(&mut self, delta: f32);
    
    #[wasm_bindgen]
    pub fn pan(&mut self, delta_x: f32, delta_y: f32);
    
    #[wasm_bindgen]
    pub fn rotate(&mut self, delta_x: f32, delta_y: f32);
    
    #[wasm_bindgen]
    pub fn get_camera_state(&self) -> JsValue; // Returns { distance, panX, panY, rotationX, rotationY }
    
    /// Render a frame
    /// Returns true if rendering occurred (was dirty), false if skipped
    #[wasm_bindgen]
    pub fn render(&mut self) -> Result<bool, JsValue>;
    
    /// Check if a render is pending
    #[wasm_bindgen]
    pub fn is_dirty(&self) -> bool;
    
    /// Mark as dirty (forces re-render on next render() call)
    #[wasm_bindgen]
    pub fn mark_dirty(&mut self);
    
    /// Handle canvas resize
    #[wasm_bindgen]
    pub fn resize(&mut self, width: u32, height: u32);
    
    /// Get canvas dimensions
    #[wasm_bindgen]
    pub fn get_dimensions(&self) -> JsValue; // Returns { width, height }
    
    /// Cleanup
    #[wasm_bindgen]
    pub fn destroy(&mut self);
}
```

### Internal Structure

```rust
// camera.rs
pub struct Camera {
    pub distance: f32,
    pub pan_x: f32,
    pub pan_y: f32,
    pub rotation_x: f32,
    pub rotation_y: f32,
    
    width: u32,
    height: u32,
    
    // Cached matrices
    view_matrix: Matrix4<f32>,
    projection_matrix: Matrix4<f32>,
    view_proj_matrix: Matrix4<f32>,
    
    view_dirty: bool,
    projection_dirty: bool,
}

impl Camera {
    pub fn update_matrices(&mut self) -> bool; // Returns true if changed
    pub fn project_to_screen(&self, world_pos: Point3<f32>) -> (f32, f32, f32); // (screen_x, screen_y, depth)
}
```

```rust
// renderer.rs - render loop pseudocode
impl GraphCore {
    pub fn render(&mut self) -> Result<bool, JsValue> {
        if !self.dirty {
            return Ok(false);
        }
        
        // Update camera matrices
        self.camera.update_matrices();
        
        // Read world positions from shared buffer
        // Project to screen coordinates
        // Write screen positions and depths to output buffers
        
        if let (Some(world_pos), Some(screen_pos), Some(depths)) = 
            (&self.world_positions, &self.screen_positions, &self.depths) 
        {
            for i in 0..self.node_count as usize {
                let wx = world_pos.get_index((i * 3) as u32);
                let wy = world_pos.get_index((i * 3 + 1) as u32);
                let wz = world_pos.get_index((i * 3 + 2) as u32);
                
                let (sx, sy, depth) = self.camera.project_to_screen(Point3::new(wx, wy, wz));
                
                screen_pos.set_index((i * 2) as u32, sx);
                screen_pos.set_index((i * 2 + 1) as u32, sy);
                depths.set_index(i as u32, depth);
            }
        }
        
        // GPU render pass for node spheres
        // ... wgpu rendering code ...
        
        self.dirty = false;
        Ok(true)
    }
}
```

### Key Design Decisions

1. **JS owns the buffers:** JavaScript allocates the `Float32Array` and `Uint8Array` buffers and passes them to WASM. This allows zero-copy reads/writes across the boundary.

2. **Minimal API surface:** The Rust layer exposes only camera control, buffer binding, and render triggering. All graph logic lives in JavaScript layers above.

3. **Dirty flag optimisation:** `render()` returns early if nothing has changed. The JS layer can call `render()` on every animation frame without waste.

---

## Layer 2: @econic/graph-3d (TypeScript)

### Responsibility

Bridge between graph concepts and raw rendering. Manages visible nodes, HTML overlays, user interaction, and buffer orchestration. This layer does not understand graph hierarchy — it works with a flat list of visible nodes.

### Public Interface

```typescript
// index.ts
export { Graph3D } from './Graph3D';
export type {
    Graph3DOptions,
    VisibleNode,
    OverlayOptions,
    OverlayFactory,
    CameraState,
    InteractionOptions,
    Graph3DEvents,
    Vec3,
    BoundingBox,
} from './types';
```

```typescript
// types.ts

export type Vec3 = [number, number, number];
export type BoundingBox = { min: Vec3; max: Vec3 };

export interface Graph3DOptions {
    canvas: HTMLCanvasElement;
    overlayContainer?: HTMLElement;      // Defaults to canvas.parentElement
    background?: string;                 // Hex colour, default '#1a1a2e'
    initialCapacity?: number;            // Max nodes, default 10000
    camera?: Partial<CameraState>;
    interaction?: InteractionOptions;
}

export interface VisibleNode {
    id: string;
    position: Vec3;
    scale?: number;          // Default 1.0
    opacity?: number;        // Default 1.0 (used for overlay styling)
    visible?: boolean;       // Default true
    data?: unknown;          // Arbitrary data passed to overlay factory
}

export interface OverlayOptions {
    offset?: Vec3;                                           // World-space offset from node position
    anchor?: 'center' | 'top' | 'bottom' | 'left' | 'right'; // CSS transform origin
    zIndexStrategy?: 'depth' | 'fixed' | 'none';             // How to set z-index
    hideWhenOffscreen?: boolean;                             // Hide overlay when node is behind camera
}

export type OverlayFactory = (node: VisibleNode) => HTMLElement;

export interface CameraState {
    distance: number;
    panX: number;
    panY: number;
    rotationX: number;
    rotationY: number;
}

export interface InteractionOptions {
    orbit?: boolean;         // Enable orbit rotation, default true
    pan?: boolean;           // Enable panning, default true
    zoom?: boolean;          // Enable zoom, default true
    zoomSpeed?: number;      // Zoom sensitivity, default 1.0
    panSpeed?: number;       // Pan sensitivity, default 1.0
    rotateSpeed?: number;    // Rotation sensitivity, default 1.0
}

export interface Graph3DEvents {
    'frame': () => void;                                              // Fired after each render
    'cameraChange': (state: CameraState) => void;                     // Fired when camera moves
}

export interface ScreenPosition {
    x: number;
    y: number;
    depth: number;
}
```

```typescript
// Graph3D.ts

import { GraphCore } from '@econic/graph-core';
import type {
    Graph3DOptions,
    VisibleNode,
    OverlayOptions,
    OverlayFactory,
    CameraState,
    InteractionOptions,
    Graph3DEvents,
    Vec3,
    BoundingBox,
    ScreenPosition,
} from './types';

export class Graph3D {
    constructor(options: Graph3DOptions);
    
    // ─────────────────────────────────────────────────────────────
    // Node Management
    // ─────────────────────────────────────────────────────────────
    
    /** Replace all visible nodes */
    setNodes(nodes: VisibleNode[]): void;
    
    /** Update a single node's position */
    updateNodePosition(id: string, position: Vec3): void;
    
    /** Batch update multiple node positions */
    updateNodePositions(updates: Array<{ id: string; position: Vec3 }>): void;
    
    /** Update a single node's scale */
    updateNodeScale(id: string, scale: number): void;
    
    /** Update a single node's opacity (affects overlay styling) */
    updateNodeOpacity(id: string, opacity: number): void;
    
    /** Remove a single node */
    removeNode(id: string): void;
    
    /** Remove all nodes */
    clear(): void;
    
    /** Get current node count */
    getNodeCount(): number;
    
    // ─────────────────────────────────────────────────────────────
    // Batch Updates (for animation frames)
    // ─────────────────────────────────────────────────────────────
    
    /** Begin a batch update - suspends rendering until endBatch() */
    beginBatch(): void;
    
    /** End batch update - commits changes and triggers render */
    endBatch(): void;
    
    // ─────────────────────────────────────────────────────────────
    // Overlays
    // ─────────────────────────────────────────────────────────────
    
    /** Attach an HTML element as overlay for a node */
    attachOverlay(nodeId: string, element: HTMLElement, options?: OverlayOptions): void;
    
    /** Attach a factory function that creates overlay elements */
    attachOverlayFactory(nodeId: string, factory: OverlayFactory, options?: OverlayOptions): void;
    
    /** Remove overlay for a node */
    detachOverlay(nodeId: string): void;
    
    /** Get the overlay element for a node (if any) */
    getOverlayElement(nodeId: string): HTMLElement | null;
    
    /** Detach all overlays */
    clearOverlays(): void;
    
    // ─────────────────────────────────────────────────────────────
    // Camera
    // ─────────────────────────────────────────────────────────────
    
    /** Set camera state directly */
    setCameraState(state: Partial<CameraState>): void;
    
    /** Get current camera state */
    getCameraState(): CameraState;
    
    /** Reset camera to default position */
    resetCamera(): void;
    
    /** Animate camera to frame the given bounds */
    frameBounds(bounds: BoundingBox, options?: { padding?: number; animate?: boolean; duration?: number }): void;
    
    /** Frame all current nodes */
    frameAll(options?: { padding?: number; animate?: boolean; duration?: number }): void;
    
    // ─────────────────────────────────────────────────────────────
    // Interaction
    // ─────────────────────────────────────────────────────────────
    
    /** Enable user interaction (orbit, pan, zoom) */
    enableInteraction(options?: InteractionOptions): void;
    
    /** Disable user interaction */
    disableInteraction(): void;
    
    /** Check if interaction is enabled */
    isInteractionEnabled(): boolean;
    
    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────
    
    /** Subscribe to an event */
    on<K extends keyof Graph3DEvents>(event: K, handler: Graph3DEvents[K]): void;
    
    /** Unsubscribe from an event */
    off<K extends keyof Graph3DEvents>(event: K, handler: Graph3DEvents[K]): void;
    
    // ─────────────────────────────────────────────────────────────
    // Screen Position Access
    // ─────────────────────────────────────────────────────────────
    
    /** Get screen position for a single node */
    getScreenPosition(nodeId: string): ScreenPosition | null;
    
    /** Get screen positions for all nodes */
    getScreenPositions(): Map<string, ScreenPosition>;
    
    // ─────────────────────────────────────────────────────────────
    // Lifecycle
    // ─────────────────────────────────────────────────────────────
    
    /** Handle canvas resize */
    resize(width: number, height: number): void;
    
    /** Start the render loop */
    start(): void;
    
    /** Stop the render loop */
    stop(): void;
    
    /** Clean up all resources */
    destroy(): void;
}
```

### Internal Implementation Sketch

```typescript
// BufferManager.ts

export class BufferManager {
    private capacity: number;
    
    // Input buffers (JS writes, WASM reads)
    public worldPositions: Float32Array;  // [x, y, z, ...] * capacity
    public scales: Float32Array;          // [s, ...] * capacity
    public visibility: Uint8Array;        // [0|1, ...] * capacity
    
    // Output buffers (WASM writes, JS reads)
    public screenPositions: Float32Array; // [x, y, ...] * capacity
    public depths: Float32Array;          // [d, ...] * capacity
    
    constructor(capacity: number) {
        this.capacity = capacity;
        this.worldPositions = new Float32Array(capacity * 3);
        this.scales = new Float32Array(capacity);
        this.visibility = new Uint8Array(capacity);
        this.screenPositions = new Float32Array(capacity * 2);
        this.depths = new Float32Array(capacity);
    }
    
    setPosition(index: number, x: number, y: number, z: number): void {
        const offset = index * 3;
        this.worldPositions[offset] = x;
        this.worldPositions[offset + 1] = y;
        this.worldPositions[offset + 2] = z;
    }
    
    getScreenPosition(index: number): { x: number; y: number; depth: number } {
        return {
            x: this.screenPositions[index * 2],
            y: this.screenPositions[index * 2 + 1],
            depth: this.depths[index],
        };
    }
    
    resize(newCapacity: number): void {
        // Reallocate buffers, copy existing data
        // ... implementation
    }
}
```

```typescript
// OverlayManager.ts

interface OverlayEntry {
    element: HTMLElement;
    options: Required<OverlayOptions>;
    nodeIndex: number;
}

export class OverlayManager {
    private container: HTMLElement;
    private overlays: Map<string, OverlayEntry> = new Map();
    private nodeIndexById: Map<string, number>;
    private buffers: BufferManager;
    
    constructor(container: HTMLElement, buffers: BufferManager, nodeIndexById: Map<string, number>) {
        this.container = container;
        this.buffers = buffers;
        this.nodeIndexById = nodeIndexById;
    }
    
    attach(nodeId: string, element: HTMLElement, options: OverlayOptions = {}): void {
        const index = this.nodeIndexById.get(nodeId);
        if (index === undefined) return;
        
        const entry: OverlayEntry = {
            element,
            options: {
                offset: options.offset ?? [0, 0, 0],
                anchor: options.anchor ?? 'center',
                zIndexStrategy: options.zIndexStrategy ?? 'depth',
                hideWhenOffscreen: options.hideWhenOffscreen ?? true,
            },
            nodeIndex: index,
        };
        
        // Style for GPU-accelerated transforms
        element.style.position = 'absolute';
        element.style.left = '0';
        element.style.top = '0';
        element.style.willChange = 'transform';
        
        this.container.appendChild(element);
        this.overlays.set(nodeId, entry);
    }
    
    detach(nodeId: string): void {
        const entry = this.overlays.get(nodeId);
        if (entry) {
            entry.element.remove();
            this.overlays.delete(nodeId);
        }
    }
    
    update(): void {
        for (const [nodeId, entry] of this.overlays) {
            const { x, y, depth } = this.buffers.getScreenPosition(entry.nodeIndex);
            
            // Hide if behind camera (depth > 1)
            if (entry.options.hideWhenOffscreen && depth > 1) {
                entry.element.style.display = 'none';
                continue;
            }
            
            entry.element.style.display = '';
            
            // Apply transform
            entry.element.style.transform = `translate3d(${x}px, ${y}px, 0)`;
            
            // Apply z-index based on depth
            if (entry.options.zIndexStrategy === 'depth') {
                entry.element.style.zIndex = String(Math.floor((1 - depth) * 10000));
            }
        }
    }
    
    updateNodeIndices(nodeIndexById: Map<string, number>): void {
        this.nodeIndexById = nodeIndexById;
        for (const [nodeId, entry] of this.overlays) {
            const newIndex = nodeIndexById.get(nodeId);
            if (newIndex !== undefined) {
                entry.nodeIndex = newIndex;
            }
        }
    }
    
    clear(): void {
        for (const entry of this.overlays.values()) {
            entry.element.remove();
        }
        this.overlays.clear();
    }
}
```

```typescript
// Graph3D.ts - simplified implementation

export class Graph3D {
    private core: GraphCore;
    private buffers: BufferManager;
    private overlays: OverlayManager;
    private nodeIndexById: Map<string, number> = new Map();
    private nodes: Map<string, VisibleNode> = new Map();
    
    private batchDepth: number = 0;
    private animationFrameId: number | null = null;
    private running: boolean = false;
    
    private eventHandlers: Map<string, Set<Function>> = new Map();
    
    constructor(options: Graph3DOptions) {
        // Initialise core renderer
        this.core = new GraphCore(options.canvas, {
            background: options.background ?? '#1a1a2e',
        });
        
        // Initialise buffers
        const capacity = options.initialCapacity ?? 10000;
        this.buffers = new BufferManager(capacity);
        
        // Register buffers with core
        this.core.set_buffers(
            this.buffers.worldPositions,
            this.buffers.scales,
            this.buffers.visibility,
            this.buffers.screenPositions,
            this.buffers.depths,
        );
        
        // Initialise overlay manager
        const overlayContainer = options.overlayContainer ?? options.canvas.parentElement!;
        this.overlays = new OverlayManager(overlayContainer, this.buffers, this.nodeIndexById);
        
        // Set initial camera state
        if (options.camera) {
            this.setCameraState(options.camera);
        }
        
        // Enable interaction if specified
        if (options.interaction) {
            this.enableInteraction(options.interaction);
        }
    }
    
    setNodes(nodes: VisibleNode[]): void {
        this.nodes.clear();
        this.nodeIndexById.clear();
        
        nodes.forEach((node, index) => {
            this.nodes.set(node.id, node);
            this.nodeIndexById.set(node.id, index);
            
            // Write to buffers
            this.buffers.setPosition(index, ...node.position);
            this.buffers.scales[index] = node.scale ?? 1.0;
            this.buffers.visibility[index] = (node.visible ?? true) ? 1 : 0;
        });
        
        this.core.set_node_count(nodes.length);
        this.overlays.updateNodeIndices(this.nodeIndexById);
        
        if (this.batchDepth === 0) {
            this.core.mark_dirty();
        }
    }
    
    beginBatch(): void {
        this.batchDepth++;
    }
    
    endBatch(): void {
        this.batchDepth--;
        if (this.batchDepth === 0) {
            this.core.mark_dirty();
        }
    }
    
    start(): void {
        if (this.running) return;
        this.running = true;
        this.tick();
    }
    
    stop(): void {
        this.running = false;
        if (this.animationFrameId !== null) {
            cancelAnimationFrame(this.animationFrameId);
            this.animationFrameId = null;
        }
    }
    
    private tick = (): void => {
        if (!this.running) return;
        
        const didRender = this.core.render();
        
        if (didRender) {
            this.overlays.update();
            this.emit('frame');
        }
        
        this.animationFrameId = requestAnimationFrame(this.tick);
    };
    
    private emit<K extends keyof Graph3DEvents>(event: K, ...args: Parameters<Graph3DEvents[K]>): void {
        const handlers = this.eventHandlers.get(event);
        if (handlers) {
            for (const handler of handlers) {
                (handler as Function)(...args);
            }
        }
    }
    
    // ... remaining method implementations
}
```

---

## Layer 3: @econic/graph-morph (TypeScript)

### Responsibility

Meta-graph management, projection to visible graph, animated transitions, and layout algorithms. This layer understands graph hierarchy and drives Layer 2 with position updates.

### Core Concepts

**Meta-Graph:** A tree structure containing all nodes (both leaf nodes and group nodes). Group nodes can be expanded or collapsed. The meta-graph is the source of truth for structure.

**Projection:** The visible graph is derived by traversing the meta-graph and collecting nodes based on expansion state. Collapsed groups appear as single nodes; expanded groups show their children.

**Transitions:** Changes to expansion state trigger animated transitions. Nodes entering the visible graph animate from their parent's position. Nodes exiting animate toward their parent's position.

### Public Interface

```typescript
// index.ts
export { GraphMorph } from './GraphMorph';
export type {
    MetaNodeDefinition,
    MetaNode,
    MetaEdgeDefinition,
    TransitionOptions,
    ExpansionState,
    LayoutAlgorithm,
    LayoutOptions,
    VisibleNode,
    VisibleEdge,
    GraphMorphEvents,
    GraphMorphState,
    EasingFunction,
} from './types';
```

```typescript
// types.ts

export type Vec3 = [number, number, number];

// ─────────────────────────────────────────────────────────────────
// Meta-Graph Definitions
// ─────────────────────────────────────────────────────────────────

export interface MetaNodeDefinition {
    id: string;
    type: 'leaf' | 'group';
    parent?: string;                    // Parent node ID (null for roots)
    children?: string[];                // Child node IDs (alternative to parent refs)
    position?: Vec3 | 'centroid';       // 'centroid' derives from children
    data?: unknown;                     // Arbitrary data passed to overlays
}

export interface MetaNode {
    readonly id: string;
    readonly type: 'leaf' | 'group';
    readonly parentId: string | null;
    readonly childIds: ReadonlySet<string>;
    readonly position: Vec3 | null;
    readonly positionMode: 'explicit' | 'centroid';
    readonly data: unknown;
    
    // Computed properties
    readonly depth: number;             // Distance from root
    readonly descendantCount: number;   // Number of leaf descendants
}

export interface MetaEdgeDefinition {
    from: string;
    to: string;
    data?: unknown;
}

// ─────────────────────────────────────────────────────────────────
// Transitions
// ─────────────────────────────────────────────────────────────────

export interface TransitionOptions {
    duration?: number;                  // Milliseconds, default 300
    easing?: EasingFunction;            // Default 'easeInOutCubic'
    stagger?: number;                   // Ms delay between child animations
    onProgress?: (t: number) => void;   // Progress callback (0 to 1)
}

export type EasingFunction =
    | 'linear'
    | 'easeIn' | 'easeOut' | 'easeInOut'
    | 'easeInCubic' | 'easeOutCubic' | 'easeInOutCubic'
    | 'easeInQuad' | 'easeOutQuad' | 'easeInOutQuad'
    | 'easeInElastic' | 'easeOutElastic'
    | ((t: number) => number);          // Custom easing function

export interface ExpansionState {
    expanded: string[];                 // IDs of expanded group nodes
}

// ─────────────────────────────────────────────────────────────────
// Layouts
// ─────────────────────────────────────────────────────────────────

export type LayoutAlgorithm =
    | 'force-directed'
    | 'hierarchical'
    | 'radial'
    | 'grid'
    | LayoutFunction;

export type LayoutFunction = (
    nodes: LayoutNode[],
    edges: LayoutEdge[],
    options: LayoutOptions
) => Map<string, Vec3>;

export interface LayoutNode {
    id: string;
    position: Vec3;
    fixed?: boolean;                    // If true, layout won't move this node
}

export interface LayoutEdge {
    from: string;
    to: string;
}

export interface LayoutOptions {
    scope?: string[];                   // Node IDs to layout (default: all visible)
    animate?: boolean;                  // Animate to new positions (default: true)
    duration?: number;                  // Animation duration
    
    // Force-directed options
    strength?: number;                  // Repulsion strength
    linkDistance?: number;              // Ideal edge length
    iterations?: number;                // Simulation iterations
    
    // Hierarchical options
    direction?: 'top-down' | 'bottom-up' | 'left-right' | 'right-left';
    levelSeparation?: number;           // Vertical spacing between levels
    nodeSeparation?: number;            // Horizontal spacing between siblings
    
    // Radial options
    centerNode?: string;                // Node at centre
    radiusStep?: number;                // Radius increment per level
    
    // Grid options
    columns?: number;                   // Number of columns
    cellSize?: number;                  // Cell size
}

// ─────────────────────────────────────────────────────────────────
// Visible Graph (Projection Output)
// ─────────────────────────────────────────────────────────────────

export interface VisibleNode {
    id: string;
    metaNodeId: string;                 // ID in meta-graph (same as id for now)
    position: Vec3;
    scale: number;
    opacity: number;
    data?: unknown;
    representsCount?: number;           // Leaf count for collapsed groups
}

export interface VisibleEdge {
    from: string;
    to: string;
    underlyingCount: number;            // Number of meta-edges this represents
    data?: unknown;
}

// ─────────────────────────────────────────────────────────────────
// Events
// ─────────────────────────────────────────────────────────────────

export interface GraphMorphEvents {
    'beforeExpand': (id: string) => void;
    'afterExpand': (id: string) => void;
    'beforeCollapse': (id: string) => void;
    'afterCollapse': (id: string) => void;
    'transitionStart': () => void;
    'transitionEnd': () => void;
    'visibleGraphChanged': (nodes: VisibleNode[], edges: VisibleEdge[]) => void;
    'structureChanged': () => void;
}

// ─────────────────────────────────────────────────────────────────
// Serialisation
// ─────────────────────────────────────────────────────────────────

export interface GraphMorphState {
    nodes: MetaNodeDefinition[];
    edges: MetaEdgeDefinition[];
    expanded: string[];
    positions: Record<string, Vec3>;
}
```

```typescript
// GraphMorph.ts

import { Graph3D } from '@econic/graph-3d';
import type {
    MetaNodeDefinition,
    MetaNode,
    MetaEdgeDefinition,
    TransitionOptions,
    ExpansionState,
    LayoutAlgorithm,
    LayoutOptions,
    VisibleNode,
    VisibleEdge,
    GraphMorphEvents,
    GraphMorphState,
    Vec3,
} from './types';

export class GraphMorph {
    constructor(renderer: Graph3D, options?: GraphMorphOptions);
    
    // ─────────────────────────────────────────────────────────────
    // Meta-Graph Definition
    // ─────────────────────────────────────────────────────────────
    
    /** Define a single node */
    defineNode(node: MetaNodeDefinition): void;
    
    /** Define multiple nodes at once */
    defineNodes(nodes: MetaNodeDefinition[]): void;
    
    /** Define a single edge */
    defineEdge(edge: MetaEdgeDefinition): void;
    
    /** Define multiple edges at once */
    defineEdges(edges: MetaEdgeDefinition[]): void;
    
    /** Clear all nodes and edges */
    clear(): void;
    
    // ─────────────────────────────────────────────────────────────
    // Dynamic Structure Changes
    // ─────────────────────────────────────────────────────────────
    
    /** Add a node to the meta-graph */
    addNode(node: MetaNodeDefinition): void;
    
    /** Remove a node (and reparent its children to grandparent) */
    removeNode(id: string): void;
    
    /** Move a node to a new parent */
    reparentNode(id: string, newParentId: string | null): void;
    
    /** Add an edge */
    addEdge(edge: MetaEdgeDefinition): void;
    
    /** Remove an edge */
    removeEdge(fromId: string, toId: string): void;
    
    // ─────────────────────────────────────────────────────────────
    // Node Queries
    // ─────────────────────────────────────────────────────────────
    
    /** Get a node by ID */
    getNode(id: string): MetaNode | null;
    
    /** Get all children of a node */
    getChildren(id: string): MetaNode[];
    
    /** Get parent of a node */
    getParent(id: string): MetaNode | null;
    
    /** Get all ancestors (parent, grandparent, ...) */
    getAncestors(id: string): MetaNode[];
    
    /** Get all descendants (children, grandchildren, ...) */
    getDescendants(id: string): MetaNode[];
    
    /** Get all root nodes (nodes with no parent) */
    getRoots(): MetaNode[];
    
    /** Get all nodes */
    getAllNodes(): MetaNode[];
    
    /** Get all edges */
    getAllEdges(): MetaEdgeDefinition[];
    
    // ─────────────────────────────────────────────────────────────
    // Expansion State
    // ─────────────────────────────────────────────────────────────
    
    /** Check if a group node is expanded */
    isExpanded(id: string): boolean;
    
    /** Get all expanded node IDs */
    getExpandedIds(): Set<string>;
    
    /** Set expansion state immediately (no animation) */
    setExpanded(ids: string[]): void;
    
    // ─────────────────────────────────────────────────────────────
    // Transitions (Animated)
    // ─────────────────────────────────────────────────────────────
    
    /** Expand a group node */
    expand(id: string, options?: TransitionOptions): Promise<void>;
    
    /** Collapse a group node */
    collapse(id: string, options?: TransitionOptions): Promise<void>;
    
    /** Toggle a group node's expansion state */
    toggle(id: string, options?: TransitionOptions): Promise<void>;
    
    /** Expand all group nodes */
    expandAll(options?: TransitionOptions): Promise<void>;
    
    /** Collapse all group nodes */
    collapseAll(options?: TransitionOptions): Promise<void>;
    
    /** Transition to a specific expansion state */
    transitionTo(state: ExpansionState, options?: TransitionOptions): Promise<void>;
    
    // ─────────────────────────────────────────────────────────────
    // Position Management
    // ─────────────────────────────────────────────────────────────
    
    /** Set a node's position explicitly */
    setNodePosition(id: string, position: Vec3): void;
    
    /** Get a node's current position */
    getNodePosition(id: string): Vec3 | null;
    
    /** Set multiple node positions */
    setNodePositions(positions: Record<string, Vec3>): void;
    
    // ─────────────────────────────────────────────────────────────
    // Layout Algorithms
    // ─────────────────────────────────────────────────────────────
    
    /** Apply a layout algorithm to position nodes */
    applyLayout(algorithm: LayoutAlgorithm, options?: LayoutOptions): Promise<void>;
    
    // ─────────────────────────────────────────────────────────────
    // Visible Graph Queries
    // ─────────────────────────────────────────────────────────────
    
    /** Get currently visible nodes */
    getVisibleNodes(): VisibleNode[];
    
    /** Get currently visible edges */
    getVisibleEdges(): VisibleEdge[];
    
    /** Check if a specific node is currently visible */
    isVisible(id: string): boolean;
    
    /** Get the visible ancestor of a node (which visible node represents it) */
    getVisibleAncestor(id: string): string | null;
    
    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────
    
    /** Subscribe to an event */
    on<K extends keyof GraphMorphEvents>(event: K, handler: GraphMorphEvents[K]): void;
    
    /** Unsubscribe from an event */
    off<K extends keyof GraphMorphEvents>(event: K, handler: GraphMorphEvents[K]): void;
    
    // ─────────────────────────────────────────────────────────────
    // Serialisation
    // ─────────────────────────────────────────────────────────────
    
    /** Export current state to JSON-serialisable object */
    exportState(): GraphMorphState;
    
    /** Import state from serialised object */
    importState(state: GraphMorphState): void;
}

export interface GraphMorphOptions {
    // Future options
}
```

### Internal Implementation

```typescript
// MetaGraphStore.ts

interface InternalMetaNode {
    id: string;
    type: 'leaf' | 'group';
    parentId: string | null;
    childIds: Set<string>;
    position: Vec3 | null;
    positionMode: 'explicit' | 'centroid';
    data: unknown;
    depth: number;
    descendantCount: number;
}

export class MetaGraphStore {
    private nodes: Map<string, InternalMetaNode> = new Map();
    private edges: Map<string, Set<string>> = new Map();      // from → Set<to>
    private reverseEdges: Map<string, Set<string>> = new Map(); // to → Set<from>
    private roots: Set<string> = new Set();
    
    addNode(def: MetaNodeDefinition): void {
        const node: InternalMetaNode = {
            id: def.id,
            type: def.type,
            parentId: def.parent ?? null,
            childIds: new Set(def.children ?? []),
            position: def.position === 'centroid' ? null : (def.position ?? null),
            positionMode: def.position === 'centroid' ? 'centroid' : 'explicit',
            data: def.data,
            depth: 0,
            descendantCount: 0,
        };
        
        this.nodes.set(def.id, node);
        
        if (node.parentId) {
            const parent = this.nodes.get(node.parentId);
            if (parent) {
                parent.childIds.add(def.id);
            }
        } else {
            this.roots.add(def.id);
        }
        
        this.recompute();
    }
    
    removeNode(id: string): void {
        const node = this.nodes.get(id);
        if (!node) return;
        
        // Reparent children
        for (const childId of node.childIds) {
            const child = this.nodes.get(childId);
            if (child) {
                child.parentId = node.parentId;
                if (node.parentId) {
                    this.nodes.get(node.parentId)?.childIds.add(childId);
                } else {
                    this.roots.add(childId);
                }
            }
        }
        
        // Remove from parent
        if (node.parentId) {
            this.nodes.get(node.parentId)?.childIds.delete(id);
        } else {
            this.roots.delete(id);
        }
        
        // Remove edges
        this.edges.delete(id);
        this.reverseEdges.delete(id);
        for (const tos of this.edges.values()) tos.delete(id);
        for (const froms of this.reverseEdges.values()) froms.delete(id);
        
        this.nodes.delete(id);
        this.recompute();
    }
    
    reparentNode(id: string, newParentId: string | null): void {
        const node = this.nodes.get(id);
        if (!node) return;
        
        // Remove from old parent
        if (node.parentId) {
            this.nodes.get(node.parentId)?.childIds.delete(id);
        } else {
            this.roots.delete(id);
        }
        
        // Add to new parent
        node.parentId = newParentId;
        if (newParentId) {
            this.nodes.get(newParentId)?.childIds.add(id);
        } else {
            this.roots.add(id);
        }
        
        this.recompute();
    }
    
    private recompute(): void {
        this.recomputeDepths();
        this.recomputeDescendantCounts();
        this.recomputeCentroidPositions();
    }
    
    private recomputeDepths(): void {
        const visit = (id: string, depth: number): void => {
            const node = this.nodes.get(id);
            if (!node) return;
            node.depth = depth;
            for (const childId of node.childIds) {
                visit(childId, depth + 1);
            }
        };
        
        for (const rootId of this.roots) {
            visit(rootId, 0);
        }
    }
    
    private recomputeDescendantCounts(): void {
        const count = (id: string): number => {
            const node = this.nodes.get(id);
            if (!node) return 0;
            
            if (node.childIds.size === 0) {
                node.descendantCount = 0;
                return 1;
            }
            
            let total = 0;
            for (const childId of node.childIds) {
                total += count(childId);
            }
            node.descendantCount = total;
            return total + 1;
        };
        
        for (const rootId of this.roots) {
            count(rootId);
        }
    }
    
    private recomputeCentroidPositions(): void {
        // Bottom-up: compute centroids from children
        const computeCentroid = (id: string): Vec3 | null => {
            const node = this.nodes.get(id);
            if (!node) return null;
            
            if (node.positionMode === 'explicit') {
                return node.position;
            }
            
            if (node.childIds.size === 0) {
                return node.position;
            }
            
            let sumX = 0, sumY = 0, sumZ = 0, count = 0;
            for (const childId of node.childIds) {
                const childPos = computeCentroid(childId);
                if (childPos) {
                    sumX += childPos[0];
                    sumY += childPos[1];
                    sumZ += childPos[2];
                    count++;
                }
            }
            
            if (count > 0) {
                node.position = [sumX / count, sumY / count, sumZ / count];
            }
            
            return node.position;
        };
        
        for (const rootId of this.roots) {
            computeCentroid(rootId);
        }
    }
    
    // Query methods...
    getNode(id: string): InternalMetaNode | undefined { return this.nodes.get(id); }
    getRoots(): string[] { return Array.from(this.roots); }
    getChildren(id: string): string[] { return Array.from(this.nodes.get(id)?.childIds ?? []); }
    // ... etc
}
```

```typescript
// ProjectionEngine.ts

export class ProjectionEngine {
    private store: MetaGraphStore;
    private expanded: Set<string> = new Set();
    
    constructor(store: MetaGraphStore) {
        this.store = store;
    }
    
    setExpanded(ids: string[]): void {
        this.expanded = new Set(ids);
    }
    
    isExpanded(id: string): boolean {
        return this.expanded.has(id);
    }
    
    project(): { nodes: VisibleNode[]; edges: VisibleEdge[] } {
        const visibleNodes: VisibleNode[] = [];
        const nodeToVisibleAncestor: Map<string, string> = new Map();
        
        const traverse = (id: string): string | null => {
            const node = this.store.getNode(id);
            if (!node) return null;
            
            if (node.type === 'leaf') {
                visibleNodes.push({
                    id: node.id,
                    metaNodeId: node.id,
                    position: node.position ?? [0, 0, 0],
                    scale: 1.0,
                    opacity: 1.0,
                    data: node.data,
                });
                nodeToVisibleAncestor.set(id, id);
                return id;
            }
            
            if (node.type === 'group') {
                if (this.expanded.has(id)) {
                    // Expanded: traverse children
                    for (const childId of node.childIds) {
                        const visibleId = traverse(childId);
                        // Map all descendants to their visible representative
                    }
                    return null; // Group itself not visible
                } else {
                    // Collapsed: show group as single node
                    visibleNodes.push({
                        id: node.id,
                        metaNodeId: node.id,
                        position: node.position ?? [0, 0, 0],
                        scale: 1.0,
                        opacity: 1.0,
                        data: node.data,
                        representsCount: node.descendantCount,
                    });
                    
                    // Map this node and all descendants to this visible node
                    this.mapDescendants(id, id, nodeToVisibleAncestor);
                    return id;
                }
            }
            
            return null;
        };
        
        for (const rootId of this.store.getRoots()) {
            traverse(rootId);
        }
        
        const visibleEdges = this.deriveEdges(nodeToVisibleAncestor);
        
        return { nodes: visibleNodes, edges: visibleEdges };
    }
    
    private mapDescendants(id: string, visibleId: string, mapping: Map<string, string>): void {
        mapping.set(id, visibleId);
        const node = this.store.getNode(id);
        if (node) {
            for (const childId of node.childIds) {
                this.mapDescendants(childId, visibleId, mapping);
            }
        }
    }
    
    private deriveEdges(nodeToVisibleAncestor: Map<string, string>): VisibleEdge[] {
        const edgeCounts: Map<string, number> = new Map();
        
        for (const edge of this.store.getAllEdges()) {
            const visibleFrom = nodeToVisibleAncestor.get(edge.from);
            const visibleTo = nodeToVisibleAncestor.get(edge.to);
            
            if (!visibleFrom || !visibleTo) continue;
            if (visibleFrom === visibleTo) continue; // Internal edge
            
            const key = `${visibleFrom}→${visibleTo}`;
            edgeCounts.set(key, (edgeCounts.get(key) ?? 0) + 1);
        }
        
        const edges: VisibleEdge[] = [];
        for (const [key, count] of edgeCounts) {
            const [from, to] = key.split('→');
            edges.push({ from, to, underlyingCount: count });
        }
        
        return edges;
    }
}
```

```typescript
// TransitionEngine.ts

interface TransitionPlan {
    entering: Array<{ id: string; from: Vec3; to: Vec3 }>;
    exiting: Array<{ id: string; from: Vec3; to: Vec3 }>;
    moving: Array<{ id: string; from: Vec3; to: Vec3 }>;
}

export class TransitionEngine {
    private renderer: Graph3D;
    private store: MetaGraphStore;
    private projection: ProjectionEngine;
    
    constructor(renderer: Graph3D, store: MetaGraphStore, projection: ProjectionEngine) {
        this.renderer = renderer;
        this.store = store;
        this.projection = projection;
    }
    
    async transition(
        fromState: ExpansionState,
        toState: ExpansionState,
        options: TransitionOptions
    ): Promise<void> {
        const duration = options.duration ?? 300;
        const easing = this.resolveEasing(options.easing ?? 'easeInOutCubic');
        
        // Compute visible graphs for both states
        this.projection.setExpanded(fromState.expanded);
        const fromGraph = this.projection.project();
        
        this.projection.setExpanded(toState.expanded);
        const toGraph = this.projection.project();
        
        // Compute transition plan
        const plan = this.computePlan(fromGraph.nodes, toGraph.nodes);
        
        // Execute animation
        const startTime = performance.now();
        
        return new Promise((resolve) => {
            const tick = () => {
                const elapsed = performance.now() - startTime;
                const t = Math.min(elapsed / duration, 1);
                const easedT = easing(t);
                
                this.applyFrame(plan, easedT, toGraph.nodes);
                
                if (t < 1) {
                    requestAnimationFrame(tick);
                } else {
                    // Final state
                    this.renderer.setNodes(toGraph.nodes);
                    resolve();
                }
            };
            
            requestAnimationFrame(tick);
        });
    }
    
    private computePlan(fromNodes: VisibleNode[], toNodes: VisibleNode[]): TransitionPlan {
        const fromById = new Map(fromNodes.map(n => [n.id, n]));
        const toById = new Map(toNodes.map(n => [n.id, n]));
        
        const entering: TransitionPlan['entering'] = [];
        const exiting: TransitionPlan['exiting'] = [];
        const moving: TransitionPlan['moving'] = [];
        
        // Nodes that exist in both
        for (const [id, toNode] of toById) {
            const fromNode = fromById.get(id);
            if (fromNode) {
                moving.push({ id, from: fromNode.position, to: toNode.position });
            } else {
                // New node: animate from parent's position
                const parentPos = this.findParentPosition(id, fromById) ?? toNode.position;
                entering.push({ id, from: parentPos, to: toNode.position });
            }
        }
        
        // Nodes that are removed
        for (const [id, fromNode] of fromById) {
            if (!toById.has(id)) {
                // Removed node: animate to parent's position in new graph
                const parentPos = this.findParentPosition(id, toById) ?? fromNode.position;
                exiting.push({ id, from: fromNode.position, to: parentPos });
            }
        }
        
        return { entering, exiting, moving };
    }
    
    private findParentPosition(id: string, visibleById: Map<string, VisibleNode>): Vec3 | null {
        const node = this.store.getNode(id);
        if (!node || !node.parentId) return null;
        
        const parent = visibleById.get(node.parentId);
        if (parent) return parent.position;
        
        // Recurse up
        return this.findParentPosition(node.parentId, visibleById);
    }
    
    private applyFrame(plan: TransitionPlan, t: number, finalNodes: VisibleNode[]): void {
        this.renderer.beginBatch();
        
        const allNodes: VisibleNode[] = [];
        
        // Entering nodes: fade in, move from parent
        for (const { id, from, to } of plan.entering) {
            allNodes.push({
                id,
                metaNodeId: id,
                position: this.lerp(from, to, t),
                scale: t,
                opacity: t,
            });
        }
        
        // Exiting nodes: fade out, move to parent
        for (const { id, from, to } of plan.exiting) {
            allNodes.push({
                id,
                metaNodeId: id,
                position: this.lerp(from, to, t),
                scale: 1 - t,
                opacity: 1 - t,
            });
        }
        
        // Moving nodes: interpolate position
        for (const { id, from, to } of plan.moving) {
            const finalNode = finalNodes.find(n => n.id === id);
            allNodes.push({
                id,
                metaNodeId: id,
                position: this.lerp(from, to, t),
                scale: 1,
                opacity: 1,
                data: finalNode?.data,
            });
        }
        
        this.renderer.setNodes(allNodes);
        this.renderer.endBatch();
    }
    
    private lerp(from: Vec3, to: Vec3, t: number): Vec3 {
        return [
            from[0] + (to[0] - from[0]) * t,
            from[1] + (to[1] - from[1]) * t,
            from[2] + (to[2] - from[2]) * t,
        ];
    }
    
    private resolveEasing(easing: EasingFunction): (t: number) => number {
        if (typeof easing === 'function') return easing;
        
        const easings: Record<string, (t: number) => number> = {
            'linear': t => t,
            'easeIn': t => t * t,
            'easeOut': t => 1 - (1 - t) * (1 - t),
            'easeInOut': t => t < 0.5 ? 2 * t * t : 1 - Math.pow(-2 * t + 2, 2) / 2,
            'easeInCubic': t => t * t * t,
            'easeOutCubic': t => 1 - Math.pow(1 - t, 3),
            'easeInOutCubic': t => t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2,
            // ... more easings
        };
        
        return easings[easing] ?? easings['linear'];
    }
}
```

---

## Layer 4: @econic/graph-ui (Future)

### Responsibility

Framework-specific bindings (React, Vue, Svelte) providing declarative component APIs. Not part of MVP.

### Conceptual API (React)

```tsx
import { Graph, Node, Edge, Group } from '@econic/graph-ui/react';

function NetworkVisualization() {
    const [expanded, setExpanded] = useState(['cluster-a']);
    
    return (
        <Graph 
            camera={{ distance: 5 }}
            onNodeClick={(id) => console.log('Clicked', id)}
        >
            <Group id="cluster-a" position={[0, 0, 0]} expanded={expanded.includes('cluster-a')}>
                <Node id="a1" position={[-1, 0, 0]}>
                    <ServerPanel name="Server 1" />
                </Node>
                <Node id="a2" position={[1, 0, 0]}>
                    <ServerPanel name="Server 2" />
                </Node>
            </Group>
            
            <Node id="database" position={[0, -2, 0]}>
                <DatabasePanel />
            </Node>
            
            <Edge from="a1" to="database" />
            <Edge from="a2" to="database" />
        </Graph>
    );
}
```

---

## File Structure

```
packages/
├── graph-core/
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs              # WASM entry point, public interface
│   │   ├── renderer.rs         # WebGPU rendering logic
│   │   ├── camera.rs           # Camera transforms and projection
│   │   ├── buffers.rs          # Buffer management utilities
│   │   ├── shaders/
│   │   │   ├── node.wgsl       # Node sphere shader
│   │   │   └── line.wgsl       # Edge line shader (future)
│   │   └── utils.rs
│   ├── pkg/                    # wasm-pack output
│   │   ├── graph_core.js
│   │   ├── graph_core_bg.wasm
│   │   └── graph_core.d.ts
│   └── package.json            # NPM package wrapping WASM

├── graph-3d/
│   ├── package.json
│   ├── tsconfig.json
│   ├── src/
│   │   ├── index.ts            # Public exports
│   │   ├── Graph3D.ts          # Main class
│   │   ├── BufferManager.ts    # Shared buffer management
│   │   ├── OverlayManager.ts   # HTML overlay positioning
│   │   ├── InteractionHandler.ts # Mouse/touch interaction
│   │   ├── EventEmitter.ts     # Event system
│   │   └── types.ts            # Type definitions
│   └── dist/

├── graph-morph/
│   ├── package.json
│   ├── tsconfig.json
│   ├── src/
│   │   ├── index.ts            # Public exports
│   │   ├── GraphMorph.ts       # Main class
│   │   ├── MetaGraphStore.ts   # Tree structure storage
│   │   ├── ProjectionEngine.ts # Visible graph derivation
│   │   ├── TransitionEngine.ts # Animation orchestration
│   │   ├── layouts/
│   │   │   ├── index.ts
│   │   │   ├── force-directed.ts
│   │   │   ├── hierarchical.ts
│   │   │   ├── radial.ts
│   │   │   └── grid.ts
│   │   ├── easing.ts           # Easing functions
│   │   └── types.ts            # Type definitions
│   └── dist/

└── graph-ui/                   # Future
    ├── react/
    ├── vue/
    └── svelte/
```

---

## Development Order

1. **@econic/graph-core** — Adapt existing Rust renderer to buffer-based interface
2. **@econic/graph-3d** — Build overlay management and buffer orchestration
3. **@econic/graph-morph** — Implement meta-graph, projection, and transitions
4. **Integration** — Wire layers together, build examples
5. **@econic/graph-ui** — Framework bindings (post-MVP)

---

## Open Questions for Implementation

1. **Edge rendering detail:** When visible edges cross behind the camera, should they be clipped, hidden, or allowed to render incorrectly?

2. **Overlay lifecycle:** When a node exits during a transition (fades out), when exactly should its overlay be removed from the DOM?

3. **Layout interruptibility:** If a layout animation is in progress and the user triggers a collapse, should the layout complete first, blend, or be interrupted?

4. **Position buffer growth:** If the developer adds more nodes than `initialCapacity`, should buffers auto-resize or throw an error?

5. **Camera animation API:** Should `frameBounds` and similar methods support animation? If so, how does this interact with transition animations?

---

*Document version: 1.0*
*Last updated: November 2025*