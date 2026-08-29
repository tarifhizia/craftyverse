# craftyverse

# 🌍 Procedural Worldbuilding Engine

## Overview

Craftyverse is a Rust-native procedural planet generation engine designed for mobile-first worldbuilding, terrain simulation, and real-time rendering. It combines an icosahedral mesh foundation, recursive triangle subdivision, tectonic and climate simulation, biome assignment, hydrology, and a multi-scale LOD system to generate a coherent spherical world from a compact data model.

The project is structured around a triangular node topology, a dual-cap plan assembly, and a Vulkan-backed rendering stack. The system is optimized for low-memory, deterministic generation, and efficient GPU updates on constrained mobile hardware.

---

## Goals and constraints

- Target platform: mobile devices with limited memory, CPU budget, and battery usage.
- Topology: hex-dominant spherical mesh with 12 pentagonal irregularities.
- Core design goals:
  - O(n) or near-linear generation passes where possible.
  - Deterministic behavior across runs for reproducible world generation.
  - Minimal per-cell storage to keep memory pressure predictable.
  - Avoid expensive physics systems unless strictly necessary for terrain fidelity.
  - Keep the geometry, climate, and terrain pipelines stable under recursive subdivision and LOD transitions.

---

## High-level architecture

The generation pipeline is layered, with each stage depending on the previous one:

1. Geometry: icosahedron → subdivision → spherical projection → hex-dominant mesh.
2. Topology: triangular node graph, plan assembly, node parenting, port linkage.
3. Tectonics: plates, boundaries, crustal motion, elevation bias.
4. Elevation: deterministic noise, mountain shaping, smoothing, sea-level classification.
5. Climate: temperature, precipitation, wind belts, drainage and rain shadow patterns.
6. Biomes: vegetation and surface classification based on climate and elevation.
7. Hydrology: rivers, flow accumulation, lakes, drainage basins.
8. Mesh integrity: deduplication, edge stabilization, seam repair, LOD continuity.
9. Rendering: Vulkan-driven GPU pipeline with mobile-optimized LOD streaming.

This document merges the project vision, class definitions, and implementation constraints into one Rust-first specification.

---

## Core geometry model

### Node model

The underlying geometry is defined by a recursive triangular node structure that supports subdivision, topology links, and mirrored child generation.

```rust
#[derive(Clone, Debug)]
pub struct Node {
    pub center: Vec3,
    pub vertices: [Vec3; 3],
    pub direction_to_origin: Vec3,
    pub directions: [Vec3; 3],
    pub uvs: [Vec2; 3],
    pub children: [Option<Box<Node>>; 3],
    pub direction_set: DirectionSet,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum DirectionSet {
    First,
    Second,
}
```

### Node semantics

Each node stores:

- `center`: the centroid of the triangle.
- `vertices`: the triangle vertices `[A, B, C]`.
- `direction_to_origin`: vector from node center toward the planet origin.
- `directions`: local basis vectors `[i, j, k]`.
- `uvs`: barycentric UV coordinates `[uvA, uvB, uvC]`.
- `children`: optional references to 3 child nodes, organized by port.
- `direction_set`: whether the node uses the first or second orientation convention.

This recursive structure supports hierarchical or fractal-like spatial construction while keeping the triangular topology explicit.

### NodeOptions

```rust
#[derive(Clone, Copy, Debug)]
pub struct NodeOptions {
    pub side_length: f32,
}
```

`side_length` defines the linear base dimension of a triangle. It is used to size the initial pentagonal caps as well as later subdivisions.

---

## Node split logic

The `split()` operation is critical to the tessellation model.

### Geometric construction

For a triangle ABC, compute the midpoints:

- `M_AB = midpoint(A, B)`
- `M_BC = midpoint(B, C)`
- `M_CA = midpoint(C, A)`

These three midpoints define the inner center triangle.

The four resulting sub-triangles are:

- Corner at A: `(A, M_AB, M_CA)`
- Corner at B: `(B, M_BC, M_AB)`
- Corner at C: `(C, M_CA, M_BC)`
- Center: `(M_AB, M_BC, M_CA)`

Each new node receives a new center equivalent to the centroid of its subtriangle:

- `center_corner_a = centroid(A, M_AB, M_CA)`
- `center_corner_b = centroid(B, M_BC, M_AB)`
- `center_corner_c = centroid(C, M_CA, M_BC)`
- `center_middle = centroid(M_AB, M_BC, M_CA)`

UVs are subdivided in the same barycentric pattern:

- `uv_M_AB = midpoint(uvA, uvB)`
- `uv_M_BC = midpoint(uvB, uvC)`
- `uv_M_CA = midpoint(uvC, uvA)`

### Direction set rules

The node has two direction conventions:

#### First set

- `I` is perpendicular to AB oriented away from the center.
- `J` is perpendicular to BC oriented away from the center.
- `K` is perpendicular to CA oriented away from the center.

#### Second set

- `K` is perpendicular to AB oriented away from the center.
- `J` is perpendicular to BC oriented away from the center.
- `I` is perpendicular to CA oriented away from the center.

For each split child:

- Corner nodes inherit the parent direction set.
- The center node flips direction set.
- Perpendiculars must be computed from the child triangle edges, not the parent triangle.

### Topology rules

- Each new node has three child slots, but after `split()`, they are empty.
- `split()` does not recursively generate children.
- It returns the four new nodes to the caller for later attachment.
- There are no cross-connections between sibling nodes created by the same split pass.

### Reference specification

```rust
impl Node {
    pub fn split(&self) -> [Node; 4] {
        // compute midpoints, UV midpoints, child centers, and new direction sets
        // create four child nodes and return them in order:
        // [corner_a, corner_b, corner_c, center]
        unimplemented!()
    }
}
```

The resulting array follows the convention:

- `[node_a, node_b, node_c, node_center]`

---

## Plan model

The `Plan` structure acts as the assembly structure for a spherical cap and contains a root node that collaborates with generated neighbor relationships.

```rust
pub struct Plan {
    pub root: Node,
}

impl Plan {
    pub fn generate_nodes(&mut self, options: NodeOptions, pentagon_direction: Vec3);
}
```

### Plan generation rules

The plan begins with five triangles forming a pentagonal cap, where:

- the pentagon sides match the configured `side_length`;
- each outer triangle shares the origin as the apex;
- the triangle base is one edge of the pentagon;
- the system rotates the triangles according to the provided pentagon direction.

For each generated node:

- create the mirrored `node_j` across the base of the current triangle;
- set direction sets based on the node center relative to the origin;
- maintain topology through `node_i`, `node_j`, and `node_k` ports.

### Port rules

- `node_i` and `node_k` follow the current direction set.
- `node_j` flips direction set between parent and child.
- If a node is `First`, its mirrored `J` child becomes `Second`.
- If it is `Second`, the mirrored `J` child becomes `First`.

Child linkage is conceptually:

- child `node_i` points back to the current node as target `node_k`
- child `node_k` points back to the current node as target `node_i`
- child `node_j` points back to the current node as target `node_j`

This is the basis for maintaining the dual-cap assembly and for interconnecting cap topology across the planet.

---

## Planet model

The `Planet` is the top-level structure containing the northern and southern cap plans.

```rust
pub struct Planet {
    pub north: Plan,
    pub south: Plan,
}

impl Planet {
    pub fn generate(&mut self, options: NodeOptions);
    pub fn draw(&self);
    pub fn split(&mut self);
}
```

### Dual-cap assembly

The spherical body is assembled from two five-triangle caps joined by a middle belt of ten triangles. A full icosahedral subdivision is modeled by joining the north and south caps through the remaining unconnected directional ports.

### Planet generation rules

1. Generate the north cap with pentagon direction pointing upward.
2. Generate the south cap with pentagon direction pointing downward.
3. Use alternating interlocking between north and south nodes for the belt connection.
4. Maintain chirality by swapping direction conventions on the mirrored side.
5. Invert vertical orientation for the southern cap to keep the global orientation consistent.

### Belt linkage rules

For each base node index `m` in the north cap:

- `North[m].children[I]` connects to `South[m]`.
- `South[m].children[K]` connects back to `North[m]`.
- `North[m].children[K]` connects to `South[(m + 1) % 5]`.
- `South[(m + 1) % 5].children[I]` connects back to `North[m]`.

This creates the interstitial belt and supports the icosahedral topology while preserving directional consistency.

### Direction inversion

- South plan vectors must invert their vertical orientation relative to north.
- If north uses `First`, the mirrored south-side linkage defaults to `Second` to preserve the handedness of the mesh.

---

## Data model for simulation and rendering

### Core world cell

```rust
#[derive(Clone, Debug, Default)]
pub struct Cell {
    pub position: Vec3,
    pub neighbors: [usize; 6],
    pub elevation: f32,
    pub temperature: f32,
    pub precipitation: f32,
    pub biome: u16,
    pub plate_id: u16,
    pub is_ocean: bool,
    pub flow_to: i32,
    pub flow_accum: u32,
}
```

### Plate structure

```rust
#[derive(Clone, Debug)]
pub struct Plate {
    pub id: u16,
    pub motion: Vec2,
    pub is_continental: bool,
}
```

### Memory representation

A structure-of-arrays layout is preferred for performance:

```rust
pub struct WorldData {
    pub elevation: Vec<f32>,
    pub temperature: Vec<f32>,
    pub moisture: Vec<f32>,
    pub plate_id: Vec<u16>,
    pub biome: Vec<u16>,
    pub flow_dir: Vec<u8>,
    pub flow_accum: Vec<u32>,
}
```

This layout is efficient for simulation passes, coarse-to-fine LOD streaming, and compact GPU uploads.

---

## Tectonic simulation

The planet begins with plate seeds and drift-driven tectonic motion.

### Plate seeding

- Pick a number of random cells as plate seeds.
- Flood-fill or region expansion to assign each cell to a plate ID.
- Maintain plate influence as a coherent region on the spherical mesh.

### Plate motion

- Assign each plate a 2D motion vector on the surface manifold.
- Drift velocities influence convergence and divergence along shared boundaries.

### Boundary classification

Compare motion vectors along neighboring faces:

- convergent boundary: plates move together
- divergent boundary: plates move apart
- transform boundary: relative motion is lateral

### Base elevation generation

- Continental plates start with a higher baseline elevation.
- Oceanic plates start lower.
- Mountain chains form near convergent edges.
- Rift zones and fracture lines form near divergent edges.

This provides the initial tectonic base shape used by the later elevation and climate layers.

---

## Elevation generation

### Noise fields

The elevation stack uses layered fractal noise with increasing detail:

- large-scale noise: continental mass and planetary-scale uplift
- mid-scale noise: mountain chains, ridges, and regional variation
- small-scale noise: localized terrain detail and surface roughness

### Combined formula

```text
e = e_tect + w1 * nL + w2 * nM + w3 * nS
```

Where:

- `e_tect` is the tectonic baseline
- `nL`, `nM`, and `nS` are coarse, medium, and fine noise fields
- `w1`, `w2`, `w3` are weights controlling distribution and stability

### Sea level and smoothing

- Normalize elevation values relative to the configured base sea level.
- Mark `is_ocean` for cells below threshold.
- Apply one or two smoothing passes to reduce terrain artifacts and preserve stability.

### Mountain shaping

Mountains are formed near plate boundaries and then refined with noise-based ridge generation, slope softening, and erosion approximations. These features directly affect climate behavior and hydrology.

---

## Climate model

The climate system works from large-scale atmospheric behavior down to local precipitation patterns.

### Temperature

```text
T = T_equator - |lat| * deltaT - elevation * lapse_rate
```

Where:

- `T_equator` is the baseline equatorial temperature
- `lat` is latitude-derived angle
- `deltaT` controls the polar gradient
- `lapse_rate` reduces temperature with altitude

### Wind bands

- Equatorial zone: east-west circulation
- Subtropical zone: west-east transitions
- Mid-latitude zone: mixed flow patterns
- Polar zone: weak and unstable circulation

### Precipitation

Moisture is seeded from oceans and transported by prevailing winds. Rainfall occurs when air is forced upward over terrain such as mountains or rising warm air cells. This creates drought zones on lee sides and wet areas on windward slopes.

---

## Biome assignment

Biome classification is driven by temperature, precipitation, and elevation.

Examples include:

- tundra
- boreal forest
- temperate forest
- grassland
- desert
- savanna
- rainforest
- alpine terrain
- taiga
- coastal wetland

The classification pass should be deterministic, using thresholds for moisture and temperature bands combined with elevation and sea-level state.

---

## Hydrology

### Flow direction

Each land cell drains to its lowest neighboring cell. This creates a consistent flow graph across the surface mesh.

### Flow accumulation

Count the number of upstream cells for each cell. These values determine water concentration and river generation.

### River generation

- Cells with flow accumulation above a threshold are candidates for rivers.
- River networks should follow gravitational flow paths and be smoothed to avoid noisy branching.

### Lakes and basins

- Cells with no outlet become lakes or endorheic basins.
- Basins should be tied to local terrain minima and hydrologic connectivity.

---

## Mesh integrity and topological stability

Mesh integrity is essential for a watertight and renderable planet surface. Without strict connectivity rules, seams, duplicate vertices, and topological drift can appear.

### Common issues

- subdivision mismatch between adjacent patches
- floating-point drift causing duplicate vertices
- projection error during spherical normalization
- LOD seam gaps as mesh resolution changes

### Solutions

- vertex snapping with a fixed tolerance before deduplication
- edge registry or map-based tracking to reuse shared vertices
- seam correction pass after each LOD rebuild
- debug wireframe mode for visual inspection of topology continuity

### Implementation notes

- Deduplicate immediately after projecting vertices to the sphere.
- Keep edge registry stable across all subdivision levels.
- Run a seam stitching pass whenever transitioning LOD levels or reattaching split nodes.

---

## Procedural planet LOD architecture

The planet generation engine is structured around a multi-scale Level of Detail system that balances quality and performance.

### LOD0: Planet-scale view

Goal: represent the whole planet at low resolution.

- base mesh: subdivided icosahedron
- geometry: coarse hex/pent cell graph
- data: low-frequency elevation, plate ID, coarse climate, global biome mask
- algorithms: tectonics, global noise, coarse climate circulation
- output: stable full-world view for orbital rendering

### LOD1–3: Regional detail

Goal: refine only near active camera regions.

- each parent hex can subdivide into local patch grids
- LOD1: 7 cells
- LOD2: 19 cells
- LOD3: 37 cells
- data: local elevation, moisture, flow accumulation, regional climate
- algorithms: multi-octave noise, erosion approximation, river refinement
- output: smooth regional fidelity for atmospheric and orbit views

### LOD4+: Ground detail

Goal: provide close-range terrain fidelity.

- convert regional mesh into tile-local heightmaps
- 128×128 or 256×256 terrain patches
- data: micro-noise, material masks, object density maps
- algorithms: heightmap extraction, tessellation, micro-detail noise
- output: detailed near-surface terrain for exploration and simulation

### LOD selection

The engine should select the active mesh resolution per frame based on surface distance from the camera.

- compute distance to the surface
- determine required LOD band
- request generation tasks from the Rust core
- stream data into GPU buffers
- free distant data when no longer needed

---

## Rust technology stack

### Programming language

Rust is the primary language for the engine because it provides:

- memory safety without a garbage collector
- predictable performance with zero-cost abstractions
- excellent concurrency control for CPU and GPU task scheduling
- robust module isolation and deterministic generation pipelines

### Graphics API

The project uses Vulkan as the rendering backend. Vulkan is chosen for:

- explicit GPU control
- highly efficient mobile rendering pipelines
- mature shader pipeline and synchronization model
- reliable resource management for large dynamic terrain and LOD streaming

### Rust/Vulkan toolchain

Typical crates and libraries for this stack include:

- `ash` or `vulkano` for Vulkan bindings
- `wgpu` for portability and rapid experimentation
- `nalgebra` or `glam` for vector and matrix math
- `noise` / `fastrand` / custom noise wrappers for procedural generation
- `rapier3d` for optional physics support
- `rodio` for audio
- `fontdue` or `rusttype` for text rendering

### Mobile optimization

- use compact numeric types for world data
- minimize allocations in hot loops
- batch terrain updates per frame
- stream generated data asynchronously
- render only necessary LOD tiles for view coverage

---

## Rendering and native integration

### Vulkan rendering model

- static or semi-static mesh for base planet body
- dynamic patch meshes for regional LOD updates
- heightmap or tessellated terrain at near-ground detail
- persistent mapped GPU buffers for terrain attributes
- multithreaded generation with deferred uploads

### Shaders

- vertex shader: deform mesh toward spherical shell and apply noise-based altitude
- tessellation shader: refine ground detail near camera
- fragment shader: biome-based color blending, terrain material selection, atmospheric lighting

### Native UI and input

- touch input mapped through Rust abstractions
- platform-level event queries through Android NDK or iOS bridging
- gesture handling for zoom, rotation, and camera control
- virtual joystick and HUD overlays rendered through the Vulkan UI path

---

## Performance and memory strategy

The design favors stable throughput and predictable resource use:

- compact storage for terrain metadata
- avoid expensive fluid simulations in the base pipeline
- keep generation steps as coarse-to-fine staged passes
- use SoA data layouts for large world arrays
- keep band-limited terrain generation as the primary approach
- stream LOD data to avoid loading the entire global mesh at high detail

---

## Full generation pipeline summary

```text
1. Build icosahedron mesh
2. Subdivide triangles recursively
3. Project vertices to sphere surface
4. Relax and deduplicate mesh vertices
5. Construct hex-dominant spherical topology
6. Seed tectonic plates and assign drift vectors
7. Compute base elevation from plate behavior
8. Add multi-octave terrain noise
9. Normalize sea level and classify ocean/land
10. Compute temperature and precipitation
11. Derive wind bands and climate regions
12. Assign biome classes
13. Compute drainage, rivers, accumulation, and lakes
14. Build LOD patches for active camera range
15. Upload to Vulkan and render with terrain shaders
```

---

## Implementation notes and conventions

- Use clearly separated responsibilities between geometry, simulation, and rendering layers.
- Keep deterministic generation stable across runs by using fixed-seed noise and explicit ordering.
- Maintain explicit topology ports (`I`, `J`, `K`) to prevent broken neighbor relations during split and reattachment.
- Preserve chirality through the dual-cap assembly so the mesh remains consistent across north and south hemispheres.
- Treat mesh integrity, LOD continuity, and data validation as first-class engineering concerns.

---

## Summary

Craftyverse is a Rust-first procedural planet engine built around recursive triangular subdivision, dual-cap assembly, tectonic and climate simulation, and Vulkan-based rendering. The result is a scalable and mobile-friendly architecture for generating large, coherent spherical worlds with controlled memory usage, deterministic output, and a clear path from global terrain generation to local high-detail surface simulation.

This unified specification combines the geometric model, world-generation logic, mobile rendering platform, and multi-scale planet architecture into a single technical reference for the project.
