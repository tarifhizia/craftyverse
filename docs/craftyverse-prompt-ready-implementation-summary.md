# Craftyverse — Prompt‑Ready Implementation Summary

Build the planet generator in strictly ordered phases. Each phase must be provably closed before advancing. Fix the data model first, topology second, surface third.

## Phase 0 — Data Model

- Use 4 child slots + 3 neighbour ports; never mix relations.
- Store all nodes in an arena; links are `u32` indices with a sentinel.
- Geometry is double‑precision end‑to‑end; narrow only at GPU upload.
- Keep one struct‑of‑arrays world state; AoS is read‑only.
- Neighbour lists use 6 indices + count + sentinel; exactly 12 pentagons report 5 neighbours.

## Phase 1 — Node Primitive

- Node frame (centre, outward, ports) is always derived from vertices.
- Port‑to‑edge mapping fixed for both direction sets; assert perpendicularity + outwardness.
- Subdivision emits 4 children in fixed order; centre child flips direction set; no sibling links.

## Phase 2 — Topology

- Vertex registry with radius‑relative tolerance; weld duplicates.
- Mirror across plane through origin + base edge; mirrored vertices must lie on sphere.
- Edge‑registry linking pass per level; keys with 2 entries yield reciprocal links.
- Build hemispherical caps (5 triangles each); south ring offset by half segment.
- Mirror J‑children to derive belt; weld; link via registry.
- Belt linkage: I→K and K→I with empirically derived preceding offset.
- Per‑level pipeline: subdivide → project → snap → recompute frame → link → assert closure.
- Determinism gate: regenerate twice; hashes must match.

## Phase 3 — Cells & Substrate

- Freeze leaf cells in deterministic order; build neighbour lists from ports.
- Seed plates deterministically; grow all plates simultaneously by BFS.
- Classify boundaries (convergent/divergent/transform) from motion vectors.

## Phase 4 — Elevation

- Tectonic base: continental uplift, trenches, divergent ridges.
- Add multi‑octave 3D noise; never sample lat/long.
- Smooth; set sea level by target ocean coverage.

## Phase 5 — Climate

- Temperature = latitude falloff + elevation lapse rate.
- Wind bands: trades, westerlies, polar easterlies.
- Moisture BFS inland along wind; decay; rain shadow.
- Precipitation = moisture × warmth × latitude band factors.

## Phase 6 — Biomes & Hydrology

- Ordered biome classification; alpine override.
- Flow direction = lowest neighbour; accumulation sorted by elevation; sinks handled.
- Rivers = accumulation threshold scaled by cell count.

## Phase 7 — Render Mesh & LOD

- Build mesh from canonical vertices; no seams.
- Relief displacement per canonical vertex, not per triangle.
- Assert LOD ring counts (7, 19, 37…).
- Finer LOD vertices must lie on coarser edges; verify watertightness.

## Phase 8 — Permanent Invariants

- Promote all acceptance conditions into test suite; run at multiple depths.
- Three outcomes: pass, fail, advisory (documented).
- Budget gate: generation, simulation, validation must stay under target.

## Phase 9 — Native Port

- Port the validated model, not the original spec.
- Port test suite first; reproduce geometry hashes exactly.

## Implementation Discipline

1. Close each phase with explicit validation.
2. Do not begin the next phase until the previous one passes in a deterministic test run.
3. Treat every invariant as a gate, not a suggestion.
4. Preserve geometry, topology, and simulation checks as first-class project requirements.
5. Only port after the canonical implementation is validated and reproducible.
