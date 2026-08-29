# Craftyverse — Implementation Plan

A step-by-step build order for the procedural planet engine, written as instructions only: what to do at each step, why it must happen in that position, and how to prove the step is finished before moving on. No code. Every correction below comes from executing the spec as written in the Craftyverse Inspector, where 75 checks run against the live geometry and simulation.

Ordering rule that governs the whole plan: **each phase must be provably closed before the next one starts.** Geometry errors are invisible until climate looks wrong, and by then you cannot tell whether the bug is in the noise, the neighbour graph, or the mirror. Fix the data model first, the topology second, the surface third.

---

## Phase 0 — Fix the data model before writing any geometry

### Step 0.1 — Split children from ports

Replace the single `children: [Option<Box<Node>>; 3]` field with two separate fields: a subdivision-children field holding **four** slots, and a neighbour-ports field holding **three** slots.

These are two different relations that the spec currently overloads into one array. Subdivision produces four children (three corners plus the centre). Topology needs exactly three ports (I, J, K). An array of three cannot hold four children, so the struct as written cannot compile against its own `split()` contract, and even if it could, writing a neighbour into a child slot would silently destroy the subdivision tree.

Done when: the struct has four child slots and three port slots, and nothing in the codebase reads a neighbour out of a child slot or vice versa.

### Step 0.2 — Replace ownership pointers with an arena and indices

Delete `Box<Node>` entirely. Store every node — all levels, both caps, the belt — in one flat growable array, and make all child and port fields plain 32-bit indices into that array. Reserve a sentinel value (the maximum u32) to mean "no link yet".

Boxed ownership assumes a tree with one owner per node. Your neighbour graph is cyclic: north links to south, children link back to parents, and belt triangles link across the hemisphere seam. A cycle cannot be expressed with unique ownership, so this is not a style preference — the graph is unrepresentable otherwise. The arena also gives you cache-friendly linear passes, trivial serialisation for deterministic replay, and stable identifiers you can log and diff between runs.

Done when: no owning pointer exists between nodes, every traversal goes through the arena, and the sentinel is checked everywhere a port is read.

### Step 0.3 — Decide the numeric contract

Use double precision for all geometry construction, subdivision, projection and vertex deduplication. Use single precision only at the very end, when packing vertex buffers for the GPU.

Single precision has roughly seven significant decimal digits. Deep subdivision repeatedly halves edge lengths and renormalises to the sphere; by the fifth or sixth level the positional error is the same order as the distance between distinct vertices, so deduplication starts merging vertices that should stay apart, or failing to merge ones that should join. That produces cracks and non-manifold edges that appear only at high detail, which is the hardest class of bug to chase.

Done when: the geometry pipeline is double precision end to end and the only narrowing happens at buffer upload.

### Step 0.4 — Collapse the duplicate cell models

Keep exactly one authoritative representation of per-cell world state: the struct-of-arrays layout, with one parallel array per attribute (plate id, elevation, temperature, precipitation, moisture, biome, flow target, flow accumulation). Delete the array-of-structs `Cell` type, or keep it strictly as a read-only view assembled on demand for debugging and inspection.

Two mutable representations of the same state will diverge the first time a simulation pass forgets to update one of them. The parallel-array layout is also what every simulation pass actually wants, since each pass reads one or two attributes across all cells.

Done when: there is a single owner of world state, and any per-cell struct is derived, never stored.

### Step 0.5 — Fix the neighbour list type

Change the per-cell neighbour list from six pointer-sized values to six 32-bit indices, add an explicit neighbour-count byte, and fill unused slots with the sentinel.

Exactly twelve cells on any icosahedral sphere are pentagons with five neighbours, not six. Without a count and a sentinel, every neighbour loop reads one garbage slot for those twelve cells. It will not crash — it will quietly corrupt twelve locations on the planet, forever, in every pass that walks neighbours.

Done when: neighbour iteration is bounded by the count, and a test asserts that exactly twelve cells report five neighbours.

---

## Phase 1 — The node primitive

### Step 1.1 — Define the node's frame as derived, never stored-and-forgotten

A node carries its three vertices, its centre, the outward direction from the origin through its centre, and its three port directions. Treat the vertices as the only source of truth and everything else as derived from them by a single recompute routine.

The spec defines the centre as the centroid of the three vertices. That is true for a flat triangle, but the moment you push vertices onto the sphere the centroid sinks below the surface and the port directions stop being tangent. If any code path stores a frame and then moves the vertices, the frame is wrong and every downstream test that uses port directions gives a misleading answer.

Done when: one routine recomputes centre, outward direction and port directions from vertices, and it is called after every operation that touches vertices.

### Step 1.2 — Bind each port to a specific edge, per direction set

Fix the port-to-edge mapping and never let it drift. For the first direction set, port I belongs to the A–B edge, port J to the B–C edge, port K to the C–A edge. For the second set, the mapping is mirrored: port K takes the A–B edge, port J keeps B–C, and port I takes C–A.

The two direction sets exist to encode triangle chirality. Half the triangles on the sphere point one way, half the other; a shared edge between them must be reachable from both sides under a consistent naming rule, otherwise linkage becomes a special case per neighbour pair. J stays on the base edge in both sets because J is the axis the mirror operation reflects across — that is what makes the mirror well defined.

Done when: for every node at every depth, each port direction is perpendicular to its own edge and points away from the node centre. Both properties should be asserted, not assumed. In the Inspector these hold to within numerical noise at every depth.

### Step 1.3 — Implement subdivision

Compute the three edge midpoints, then emit exactly four children in a fixed order: the corner triangle at A, the corner at B, the corner at C, and the centre triangle formed by the three midpoints. Each corner child is built from its original corner plus the two adjacent midpoints. The three corner children inherit the parent's direction set; the centre child flips to the other set.

The flip is not cosmetic. The centre triangle is geometrically inverted relative to its parent, so its edges meet its siblings in mirrored order; without the flip its port naming contradicts its neighbours' and linkage silently pairs the wrong edges.

Compute midpoints as flat averages here. Do not project inside subdivision — projection is a separate, later step.

Done when: four children exist in the documented order, each child's centre equals its own subtriangle's centroid, the four areas sum to the parent area, all four keep the parent's winding direction, subdivision does not recurse on its own, and no sibling-to-sibling port links are written during the split.

### Step 1.4 — Honour the "split writes no sibling links" rule, then compensate

Keep the spec's rule that subdivision itself creates no links between the children. Accept that this leaves every child with three empty ports, and plan the compensating pass in Phase 2. Do not attempt to fix it by having each child link its siblings during the split.

The rule is worth keeping because sibling linkage from inside the split cannot see across the parent boundary — the child touching the parent's outer edge has a neighbour that belongs to a different parent entirely. Any linkage written during the split is therefore incomplete by construction, and partial links are worse than none because later passes cannot tell which ports are trustworthy.

Done when: after subdivision, every port on every new node is the sentinel.

---

## Phase 2 — Topology: linking, mirroring, closure

### Step 2.1 — Build a positional vertex registry with radius-relative tolerance

Before linking anything, build a registry that maps a spatial position to a canonical vertex index, snapping any position within a tolerance of an existing entry onto that entry. Scale the tolerance to the planet radius rather than fixing it as an absolute number, and implement the lookup as a spatial hash over small cells so the search stays local instead of scanning all vertices.

Two patches computed by different arithmetic paths — a cap triangle and a mirrored belt triangle, for instance — land on the same point mathematically but differ in the last few bits. Without snapping, those become two distinct vertices, the edge between the two patches is never recognised as shared, and you get a visible crack plus a hole in the neighbour graph. The tolerance must be relative because an absolute value that is safe at radius one is either meaningless or catastrophic at planetary scale.

Done when: the registry reports a non-zero number of merges with a maximum drift far below the tolerance. In the Inspector, the base icosahedron plus the belt mirror produces ten merges with drift on the order of ten to the minus sixteen.

### Step 2.2 — Fix the mirror operation

Reflect across the plane that contains the origin and the base edge, not across the base edge inside the triangle's own plane. Build the plane normal from the base edge direction and the midpoint of that edge, normalise it, and reflect each vertex by subtracting twice its component along that normal.

An in-plane mirror is correct only for flat geometry. On a sphere it produces a triangle that lies in the parent's tangent plane and therefore leaves the surface, so the mirrored belt triangle does not share its vertices with the cap it came from and cannot be welded to it. This is one of the advisory findings deliberately left visible in the Inspector: with the naive mirror, the resulting vertices are measurably off the sphere.

Done when: every mirrored vertex sits on the sphere within tolerance and coincides with the corresponding cap vertex closely enough for the registry to weld them.

### Step 2.3 — Add an edge-registry linking pass, run once per level

After each subdivision level completes, walk every node at that level, register each of its three edges under a canonical key built from its two canonical vertex indices in sorted order, and record which node and which port produced it. Every key that collects exactly two entries yields a reciprocal link: write each node into the other's corresponding port. Any key with one entry is an open boundary; any key with three or more is a topology error.

This is the compensating pass for Step 1.4. Running it per level, immediately after that level is built, means a failure is localised to the level that introduced it. Running it only once at the end tells you the mesh is broken but not where.

Done when: the pass reports zero keys with three or more entries, zero keys with a single entry on a closed sphere, and every link is reciprocal — if A reaches B through a port, B reaches A through the corresponding port.

### Step 2.4 — Generate one hemispherical cap

Build the cap as five triangles sharing the pole vertex: each triangle takes the pole plus two consecutive vertices of a latitude ring, wrapping the last back to the first. Enforce consistent outward winding on all five as they are created. Assign the first direction set to the north cap and the second to the south, and offset the south ring's rotation by half a ring segment so the two hemispheres interlock instead of aligning face to face.

The phase offset is what makes the belt possible. With the rings aligned, the seam between hemispheres is a flat ring of quadrilaterals; offset by half a segment, it becomes ten triangles that close the surface with the same primitive used everywhere else.

Done when: five triangles exist, all wound outward, sharing exactly one pole vertex, and the ring closes back on itself.

### Step 2.5 — Assemble both caps and derive the belt

Instantiate the north cap with the first direction set and the south with the second, then produce the ten belt triangles by mirroring each cap node's J child across the plane from Step 2.2. Weld everything through the vertex registry, then run the edge-registry link pass over the complete set.

Done when: the assembled surface is a closed icosahedron of twenty faces with no open edges.

### Step 2.6 — Correct the belt linkage rule and derive it rather than hard-coding it

Apply belt linkage to the **mirrored J children**, not to the cap nodes themselves. Then verify the pairing empirically instead of trusting the written rule: link each northern belt triangle's I port to the matching southern triangle's K port, and its K port to the **preceding** southern triangle's I port.

Two corrections here, both confirmed by the Inspector against live geometry. First, the spec states the rule on cap nodes, but a cap node's I and K edges are meridians running from the pole to the ring — they are shared with the two neighbouring cap nodes in the same hemisphere and never with the opposite hemisphere. Applying the rule there links the wrong pairs. Only the mirrored J children actually straddle the seam. Second, the index offset in the spec advances by one; the measured offset in the closed mesh goes back by one. The I-to-K and K-to-I port pattern itself is correct.

Do not hard-code the offset. Derive it from the edge registry, then assert that the result matches the expected pattern. That way a future change to ring phase or winding produces a failing assertion rather than a seam that is subtly wrong.

Done when: all ten belt triangles are linked with no open ports across the seam, and the derived offset is asserted rather than assumed.

### Step 2.7 — Subdivide with the correct per-level order

For each level, repeat this exact sequence: subdivide every current leaf; project all new vertices onto the sphere; snap all vertices through the registry; recompute every node's frame; run the edge-registry link pass; assert closure. Only then advance to the next level.

The order matters at every position. Projection must follow subdivision because midpoints are computed flat. Snapping must follow projection because projection is where the last-bit divergence appears. Frame recomputation must follow snapping because snapping moves vertices. Linking must follow snapping because edge keys are built from canonical vertex indices. Asserting last is what makes each level's failure attributable to that level.

Done when: at every depth the face count equals twenty times four to the depth, vertices equal ten times four to the depth plus two, edges equal thirty times four to the depth, vertices minus edges plus faces equals two, and exactly twelve vertices have degree five. The Inspector confirms all of these through depth five, at 20,480 faces.

### Step 2.8 — Close the phase with a determinism gate

Generate the planet twice from identical parameters, hash the full vertex and topology state each time, and require the hashes to match. Make this a permanent test, not a one-time check.

Determinism is the property that makes every later bug reproducible and every world shareable by seed alone. It is also the property most easily lost by accident — one iteration over a hash map, one parallel reduction with non-fixed ordering, one place where narrowing to single precision slipped in. Catch it here, where the surface area is still small.

Done when: repeated generation is bit-identical, and the same holds after any change to the geometry pipeline.

---

## Phase 3 — Cells and the simulation substrate

### Step 3.1 — Freeze the cell array from the finished leaves

Once the deepest level is closed, walk the leaves in a fixed deterministic order and assign each one a stable cell index. Build the per-cell neighbour lists from the port links established in Phase 2, applying the count-and-sentinel convention from Step 0.5. Precompute and store per-cell centre position and latitude.

Iteration order is part of the determinism contract, because every later pass that seeds randomness or breaks ties by index depends on it.

Done when: cell count equals leaf count, every neighbour relation is symmetric, exactly twelve cells report five neighbours, and no neighbour slot inside the reported count holds the sentinel.

### Step 3.2 — Seed and grow tectonic plates

Choose plate seed cells from the deterministic random stream, then grow all plates simultaneously by breadth-first expansion across the neighbour graph until every cell is claimed. Assign each plate a continental or oceanic character and a motion direction tangent to the sphere at its own centre.

Simultaneous growth is what produces plates of comparable size with irregular, natural borders. Growing them one at a time gives the first plate the whole planet and the rest the leftovers.

Done when: every cell belongs to exactly one plate, no plate is empty, and plate regions are contiguous across the neighbour graph.

### Step 3.3 — Classify boundaries

For every pair of adjacent cells belonging to different plates, compare the two plates' motion vectors against the direction between the cells to classify the boundary as convergent, divergent, or transform, and record a stress magnitude.

This is the only place where plate motion influences terrain, so it must be a separate, inspectable step. Fold it into elevation and you lose the ability to see whether mountains are appearing because of tectonics or because of noise.

Done when: every cross-plate adjacency carries a classification and a stress value, and the three classes are all present.

---

## Phase 4 — Elevation

### Step 4.1 — Lay down the tectonic base

Set a base elevation from plate character — continental plates raised, oceanic lowered — then add uplift near convergent boundaries proportional to stress, trenches near convergent boundaries where one side is oceanic, and ridges along divergent boundaries.

Done when: mountain belts follow plate boundaries rather than appearing at random, and ocean basins correspond to oceanic plates.

### Step 4.2 — Add the noise stack

Add several octaves of coherent noise sampled from the cell's three-dimensional position on the sphere, each octave at roughly double the frequency and half the amplitude of the previous one. Expose the overall noise weight and the mountain gain as parameters.

Sample in three dimensions from the sphere position, never in two dimensions from latitude and longitude. A two-dimensional parameterisation has a singularity at each pole and a discontinuity at the wrap meridian, and both will show as visible artefacts.

Done when: terrain has detail at multiple scales, no seam is visible at any meridian, and no distortion appears at the poles.

### Step 4.3 — Smooth, then set sea level by target coverage

Run a small number of neighbour-averaging passes to remove single-cell spikes. Then, rather than picking an absolute sea level, sort or histogram the elevations and choose the level that yields the ocean fraction you asked for.

Deriving sea level from a target fraction decouples world design from noise tuning: changing the noise no longer changes how much of the planet is water, so the two parameters stop fighting each other.

Done when: the achieved ocean fraction matches the target closely — the Inspector lands within a fraction of a percent — and the value is stable across seeds.

---

## Phase 5 — Climate

### Step 5.1 — Temperature

Derive base temperature from absolute latitude with a falloff slightly steeper than linear, then subtract a lapse-rate term proportional to elevation above sea level.

A pure linear falloff gives mid-latitudes temperatures that are too cold and collapses the temperate band, which is why the first pass produced tundra where forests belong. The elevation term is what puts alpine conditions on equatorial mountains.

Done when: the equator is warm, poles are below freezing, mid-latitudes sit in a genuine temperate range, and high terrain is cold at every latitude.

### Step 5.2 — Wind bands

Assign each cell a prevailing wind direction from its latitude band, following the standard circulation pattern: easterly trades near the equator, westerlies at mid-latitudes, polar easterlies at high latitudes.

Done when: bands are symmetric about the equator and each cell has a defined wind direction.

### Step 5.3 — Moisture transport

Set moisture to maximum on all ocean cells, then propagate it inland by breadth-first traversal along the wind direction, decaying it per step. Increase the decay when moving uphill so that terrain casts a rain shadow.

Breadth-first propagation from the ocean is what makes coasts wet and continental interiors dry — the single most recognisable feature of a believable climate map. Rain shadow is what makes mountain ranges matter to biomes rather than just to the silhouette.

Done when: moisture decreases with distance from the coast, and the leeward side of ranges is measurably drier than the windward side.

### Step 5.4 — Precipitation, with circulation bands

Combine moisture with a warmth factor, then multiply by a latitude band factor: wettest in the equatorial convergence zone, driest in the subtropical high-pressure belts, moderate under the westerlies, dry again at the poles.

Without the band factor, deserts only appear in continental interiors and the planet reads as one continuous climate gradient. The subtropical dry belt is what puts deserts at the latitudes where real ones sit, and it is the difference between a plausible biome map and a muddy one. Adding this multiplier is what moved the test planet from almost no forest cover to a distribution containing rainforest, savanna, desert, grassland, temperate and boreal forest, taiga, tundra and ice in sensible proportions.

Done when: dry belts appear at subtropical latitudes independently of distance from the coast, and the equatorial band is the wettest region.

---

## Phase 6 — Biomes and hydrology

### Step 6.1 — Classify biomes

Assign each cell a biome from temperature, precipitation and elevation, handling water first — deep ocean, shallow sea — then ice by temperature, then land types by the temperature and precipitation combination, with an alpine override for high terrain.

Order the tests so that each one only sees cells the previous ones did not claim. An unordered chain of conditions produces snow in rainforests and similar contradictions.

Done when: no cell is unclassified, biome regions are spatially coherent rather than speckled, and the distribution across a few different seeds contains every biome in plausible proportions.

### Step 6.2 — Flow direction and accumulation

Give each land cell a flow target: the neighbour with the lowest elevation, or none if no neighbour is lower. Then process cells from highest to lowest, accumulating each cell's flow into its target so that accumulation is computed in a single ordered pass. Cells with no lower neighbour are endorheic sinks; either mark them as lakes or fill them and recompute.

Sorting by elevation first is what lets accumulation complete in one pass without iterating to convergence. Sinks must be handled explicitly — ignore them and rivers terminate in the middle of continents.

Done when: no flow path forms a cycle, every path terminates in the ocean or in a marked sink, and total accumulation is conserved.

### Step 6.3 — Extract rivers

Mark a cell as river where accumulated flow exceeds a threshold, and scale that threshold with cell count rather than fixing it.

A fixed threshold is tuned to one resolution and wrong at every other. Too high and there are no rivers at low detail; too low and half the planet is river at high detail.

Done when: rivers run downhill without exception, reach the sea or a lake, and their count stays sensible across the whole range of subdivision depths.

---

## Phase 7 — Render mesh and level of detail

### Step 7.1 — Build the draw mesh from canonical vertices

Emit an indexed mesh whose indices refer to the registry's canonical vertices, so that adjacent triangles share vertex records rather than duplicating them.

Done when: the vertex count matches the topological count from Step 2.7 and no seam is visible where patches meet.

### Step 7.2 — Displace relief without tearing the surface

Compute each canonical vertex's displacement once, as the average of the elevations of the cells that touch it, then apply that single value everywhere that vertex is used.

Displacing per triangle, using each triangle's own cell elevation, moves a shared vertex to a different position for each triangle that references it. The result is a mesh that looks correct at low relief and splits open as relief increases.

Done when: raising relief to its maximum produces no visible cracks.

### Step 7.3 — Define the level-of-detail bands and their cell counts

Fix the intended cell count for each detail band and assert it. The first band is a single cell with its immediate ring, the second adds the next ring, the third adds another — seven, nineteen, and thirty-seven cells respectively for the first three bands.

These are hexagonal ring totals. Writing them down as assertions turns a vague performance intent into a testable property, and catches the case where a ring walk silently drops or duplicates the pentagon cells.

Done when: each band's cell count matches its declared value exactly.

### Step 7.4 — Keep adjacent detail levels watertight

When two neighbouring patches sit at different detail levels, the finer patch must place its extra edge vertices exactly on the coarser patch's edge, so no gap can open. Verify by walking the boundary between mismatched patches and confirming that every finer vertex lies on the coarser edge within tolerance.

This is the classic terrain-LOD failure. It shows up as thin flickering cracks along the horizon and is much cheaper to prevent by construction than to patch afterwards.

Done when: no gap exists along any boundary between patches of differing detail.

---

## Phase 8 — Make the invariants permanent

### Step 8.1 — Promote every check into the test suite

Turn each acceptance condition above into an automated assertion, grouped by phase, and run the whole suite at several subdivision depths — at minimum the shallowest, one middle depth, and the deepest you intend to ship.

Some failures only appear at depth, and some only at depth zero where the pentagon cases dominate. A suite that runs at a single depth misses both ends.

Done when: the suite covers node subdivision, direction sets and local basis, cap construction and the mirrored child, port topology, mesh integrity and detail continuity, belt linkage, determinism, detail bands, and simulation invariants — and it runs at multiple depths on every change.

### Step 8.2 — Distinguish failures from advisories

Give checks three outcomes rather than two: pass, fail, and advisory. Reserve advisory for known and accepted deviations, and require every advisory to carry a written explanation of why it is acceptable.

A binary suite forces you to either delete an inconvenient check or let a permanent red state train you to ignore the output. The three known advisories on this design are the naive in-plane mirror leaving the sphere, the belt offset differing from the written specification, and the pentagon cells not fitting a fixed six-neighbour list — all findings to keep visible, not bugs to suppress.

Done when: the suite reports the three counts separately and every advisory has a documented reason.

### Step 8.3 — Add a budget gate

Record generation time, simulation time and validation time on every run, and fail the build if any exceeds its budget for the target depth.

Correctness regressions are loud; performance regressions are silent until the mobile target stops holding frame rate. For reference, the reference implementation generates 20,480 faces in roughly 1.3 seconds and validates them in roughly 2.3 seconds, single-threaded, in a browser — treat that as an upper bound your native build should comfortably beat.

Done when: timings are recorded per run and compared against declared budgets.

---

## Phase 9 — Port to the native stack, last

### Step 9.1 — Port the validated logic, not the specification

Translate the corrected model — arena and indices, split-then-project-then-snap-then-link ordering, mirrored-J belt linkage with the derived offset, double precision geometry, single cell-state layout with counted neighbours — into the Rust engine. Port the test suite first and get it passing on trivial input before porting the generator.

Porting the specification as originally written reintroduces every blocker: the struct will not compile, and if forced to compile it produces an open mesh with mislinked seams. The validated model is the one that closes.

Done when: the native suite reproduces the same counts and the same geometry hashes as the reference for identical parameters.

### Step 9.2 — Only then add the graphics and physics layers

Bring in rendering, then collision, then anything gameplay-facing. Keep the generator headless and testable, with rendering as a consumer of its output rather than a participant in it.

A generator that can only run inside a live renderer cannot be tested in continuous integration, cannot be run at ten depths in a loop, and cannot be bisected when a seam appears.

Done when: the generator runs headless with no graphics context, and the renderer consumes its output without modifying it.

---

## Summary of what changes versus the original specification

| # | Original | Corrected | Severity |
|---|---|---|---|
| 1 | Three child slots, doubling as ports | Four children plus three separate ports | Blocker |
| 2 | Boxed node ownership | Flat arena with index links | Blocker |
| 3 | Belt linkage on cap nodes, offset forward by one | Linkage on mirrored J children, offset back by one | Blocker |
| 4 | Mirror across the base edge in-plane | Reflect across the plane through the origin and the base edge | Major |
| 5 | Subdivision creates no links, and nothing compensates | Add a per-level edge-registry linking pass and assert closure | Major |
| 6 | Centre defined as the centroid | Recompute the frame after every projection | Major |
| 7 | Six pointer-sized neighbours per cell | Six 32-bit indices plus a count and a sentinel | Major |
| 8 | Both an array-of-structs and a struct-of-arrays cell model | One authoritative layout | Minor |
| 9 | Absolute deduplication tolerance | Tolerance scaled to the planet radius | Minor |
| 10 | Single precision throughout | Double precision for geometry, single only at upload | Minor |

All ten were found by executing the specification rather than reading it. Every phase gate above is already implemented and passing in the Craftyverse Inspector — 72 passing, 3 advisory, 0 failing, verified from depth zero through depth five.
