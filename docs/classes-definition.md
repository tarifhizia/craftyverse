# Classes Definition

## Node Class Definition

### Overview
The `Node` class represents a geometric structure composed of a center point, directional vectors, UV coordinates, and recursive child nodes. This design supports hierarchical or fractal-like spatial structures and connectivity between nodes.

### Structure

- **Geometry**
- `center` — Point representing the center of the node.
- `vertices` — Array of the three triangle vertices (`[A, B, C]`).
- `direction_to_origin` — Vector from the node center toward the origin.
- `directions` — Array of directional vectors (`[i, j, k]`).
- `uvs` — Array of UV coordinates (`[uvA, uvB, uvC]`).
- **Topology**
- `children` — List of child nodes (3 nodes).

### Pseudocode Representation

```
class Node {
  // --- Geometry ---
  Point center
  Point[] vertices        // [A, B, C]
  Vector direction_to_origin
  Vector[] directions     // [i, j, k]
  UV[] uvs                // [uvA, uvB, uvC]

  // --- Topology ---
  List<Node> children     //  3 nodes
}
```

### Methods

- `split()` — Splits the current node into four new nodes according to geometric construction, direction set, topology, and identity rules described below.

---

## Node.split() Method Specification

### Geometric Construction Rules

1. Compute Midpoints
Let:

- MAB = midpoint(A, B)
- MBC = midpoint(B, C)
- MCA = midpoint(C, A)
These three midpoints form the center triangle.

1. Compute New Centers
Each new node gets a center computed exactly like the parent:

- CenterCornerA = centroid(A, MAB, MCA)
- CenterCornerB = centroid(B, MBC, MAB)
- CenterCornerC = centroid(C, MCA, MBC)
- CenterMiddle = centroid(MAB, MBC, MCA)

1. UV Subdivision
UVs subdivide barycentrically:

- uvMAB = midpoint(uvA, uvB)
- uvMBC = midpoint(uvB, uvC)
- uvMCA = midpoint(uvC, uvA)
Each new node receives the corresponding UV triplet.

### Direction Set Rules

1. Direction Set Inheritance

- Corner nodes inherit the same direction set (First or Second) as the parent.
- The center node always flips direction set:

- If parent uses First, center uses Second
- If parent uses Second, center uses First
This matches your existing rule for nodeJ mirroring.

1. Direction Vector Recalculation
For each new node:

- Compute vector V = center → origin
- Apply direction set rules:

- First set:

- I ⊥ AB
- J ⊥ BC
- K ⊥ CA
- Second set:

- K ⊥ AB
- J ⊥ BC
- I ⊥ CA
Perpendiculars must be computed from the new triangle edges, not the parent.

### Topology Rules

1. Each new node has 3 children

- But after split(),  only the node center should be connected to the new created nodes A B C
- Split does not recursively generate children; it only produces the 4 new nodes.

- split() returns:

```
nodeCenter 
```
- The caller decides how to reattach the new created nodes to neighbors splitter nodes.

1. No cross​‑connections

- Split does not connect the new nodes A B C to each other.
- This avoids interfering with Plan/Planet generation rules.

### Node Identity Rules
Each new node must store:

- New triangle vertices (A, B, C)
- New center
- New UVs
- New directions
- Direction set type (First/Second)

### Full Method Specification
**Signature**

```
Node[] split()
```
**Returns**

Nodes:

- Center node connected to each node corner
**Steps**

- Compute midpoints
- Compute new UV midpoints
- Build 4 triangles
- Compute centers
- Determine direction set for each
- Compute perpendicular directions
- Create nodes
- connect nodes to center node
- Return center node 

---

## NodeOptions Class Definition

### Overview
The `NodeOptions` class defines configuration parameters related to node dimensions, specifically the lengths of its sides.

### Structure

- **Parameters**
- `sideLength` — The length of the sides of the node.

### Pseudocode Representation

```
class NodeOptions {
  Float sideLength 
}
```

### Methods
*(No additional methods defined).* 

---

## Plan Class Definition

### Overview
The `Plan` class serves as the central structure of the system, encapsulating a primary node that organizes and manages the overall plan's components, height map rendering, and node generation.

### Structure

- **Parameters**
- `node` — An instance of the `Node` class representing the primary node of the plan.

### Pseudocode Representation

```
class Plan {
  Node node
  void generateNodes(NodeOptions options, Vector pentagonDirection)
}
```

### Methods

- `generateNodes(NodeOptions options, Vector pentagonDirection )` — Creates interconnected nodes using the specified configuration options.

### generateNodes Rules

- Start by generating 5 triangles to form a pentagon (sides of pentagon equal to NodeOptions.sideLength, direction of pentagon is from props) where nodes are connected through their K vectors and[...]
- For each created node, create the child nodeJ where the new triangle is the mirror of the current triangle, mirrored along the base of the current triangle.
- For each Node, the direction set depends on the node's center and the vector from the center to the origin.
- two main direction sets exist:

- **First:**

- I = perpendicular to AB going from center
- J = perpendicular to BC going from center
- K = perpendicular to CD going from center
- **Second:**

- K = perpendicular to AB going from center
- J = perpendicular to BC going from center
- I = perpendicular to CD going from center
- The choice between First and Second depends on the node's center and the vector from center to origin.
- Child node relationships:

- child nodeI should have current node as target nodeK
- child nodeK should have current node as target nodeI
- child nodeJ should have current node as target nodeJ
- Direction inheritance:

- nodeJ always switches vector directions between First and Second (if current is First, childJ uses Second, and vice versa)
- nodeI and nodeK use the same direction set as the current node
- Triangle vertices (A, B, C), center, triangle direction (First or Second), and vector from center to origin determine the geometry of the node's triangle.
- The base of the first five triangles should be the pentagon sides and the sides of the triangle are the diameter of the pentagon , and the apex of each triangle is at the origin. This setup rot[...]
- if the pentagon direction top or take the normal first and second, if bottom first become second and second become first
- The process starts by generating 5 triangles forming a pentagon where nodes connect through their K vectors and the next node, and all nodes create the I node as a mirror cut through the triang[...]

---

## Planet Class Definition

### Overview
The `Planet` class serves as the top-level container for the global structure, encapsulating both the northern and southern `Plan` instances to represent a complete icosahedron-based spherical body.

### Structure

- **Parameters**
- `north` — An instance of the `Plan` class representing the northern hemisphere/map.
- `south` — An instance of the `Plan` class representing the southern hemisphere/map.

### Pseudocode Representation

```
class Planet {
  Plan north
  Plan south
  void generate()
  void draw()
  void split()
}
```

### Methods

- `generate()` — Generates the northern plan nodes with the pentagon direction pointing to the top and the southern plan nodes with the pentagon direction pointing to the bottom. Then connects [...]
- `draw()` — Draws both the northern and southern plans in an SVG, displaying all relevant information doubled for clarity.
- `split()` — Iterates through north plan node, calls `split()` on each node, replace current by returned node and connects the newly created nodes I j k between adjacent split nodes (nodeI, no[...]

### Generate function Rules

### **Core Principle: Dual-Pentagon Dual-Cap Assembly**
An icosahedron breaks down into two 5-triangle pentagonal caps (North and South) and a 10-triangle middle antiprismatic belt. Connecting North nodes to South nodes via their remaining unconnected[...]

```
       [ North Cap: 5 Triangles ]
        /     |     |     |     
      (I)    (K)   (I)   (K)   (I)   <-- Unconnected Outer Ports
       |      |     |     |      |
      (K)    (I)   (K)   (I)   (K)   <-- Interlocking South Ports
             |     |     |     /
       [ South Cap: 5 Triangles ]
```

### **Detailed Rules for `Planet.generate()`**

1. **Cap Generation**

- Call `north.generateNodes(options, Vector(0, 1, 0))` with the top direction vector.
- Call `south.generateNodes(options, Vector(0, -1, 0))` with the bottom direction vector.
- Both caps produce 5 base triangles (nodes 0 through 4) arranged circularly, linked internally via `J` (mirrored base) and adjacent `K`/`I` vectors.
2. **Interstitial Belt Linkage (North-to-South Alignment)**

- Every North base node Nm​ (where m∈{5,6,7,8,9}) has two open directional slots (I and K).
- Because the South cap is inverted and offset by π/5 (36°), South base nodes Sn​ interlock with North nodes in an alternating zig-zag fashion:

- **Link Rule A (Node I Connection):** Connect `North[m].children[I]` to `South[m].node`. Reciprocally, assign `South[m].children[K] = North[m].node`.
- **Link Rule B (Node K Connection):** Connect `North[m].children[K]` to `South[(m + 1) % 5].node`. Reciprocally, assign `South[(m + 1) % 5].children[I] = North[m].node`.
3. **Direction & Alignment Inversion**n

- **Vector Reversal:** South plan directional vectors (I, J, K) must invert their Z-axis (or vertical orientation parameter) relative to North.
- **Direction Set Swap:** If `North[m]` uses the **First** direction set, the corresponding mirrored connection entry on `South[m]` must default to the **Second** direction set to maintain topolo[...]

---

## Planet.split() Method Specification

### Overview
The `Planet.split()` method coordinates splitting across the north (and symmetric south) caps and reconnects the resulting split-centers to grow the antiprismatic belt between caps. Splits are performed and joined in a traversal order that prioritizes connections along the nodeI direction first; once a closed loop (back to the starting node) is completed it advances to nodeJ of the starting node and repeats. The process continues until neighbouring nodes at the next level are already split (i.e. their nexti, nextj and nextk references are non-null at the target level).

### Connection rules

- When traversing and splitting, the algorithm follows nodeI links from a starting node, splitting the current node and its nodeI target and connecting the new split-centers together.
- The connection pattern between newly created centers uses the following pointer rewiring after each pair-split operation:
  - nodeSplitedCenter.nexti.nexti = nodeTargetSplitedCenter.nextj.nextk
  - nodeSplitedCenter.nextj.nexti = nodeTargetSplitedCenter.nextk.nextk
- The traversal progresses along nexti until it completes a loop back to the start; then it begins the same traversal starting from the start node's nodeJ and repeats. The whole routine terminates when the traversal reaches a node whose nexti, nextj and nextk at the next level are already present (non-null), meaning that area has been processed.

### Signature

```
void split()
```

### Algorithm (pseudocode)

```
// Start from a northern base node (Nodenorth) that begins the split traversal
currentNode = Nodenorth

// Initialize traversal pointers
nextNode = currentNode.nexti.nexti
targetNode = currentNode.nexti

// Perform initial splits (split() returns the center node of the split)
nodeSplitedCenter = currentNode.split()
nodeTargetSplitedCenter = targetNode.split()

// Loop until the traversal reaches nodes that are already at the next split level
while not (nextNode.nexti.level == currentNode.level and nextNode.nextj.level == currentNode.level and nextNode.nextk.level == currentNode.level):
    // Rewire connections between the freshly split centers
    nodeSplitedCenter.nexti.nexti = nodeTargetSplitedCenter.nextj.nextk
    nodeSplitedCenter.nextj.nexti = nodeTargetSplitedCenter.nextk.nextk

    // Advance the window of traversal
    currentNode = nodeSplitedCenter
    nodeSplitedCenter = nodeTargetSplitedCenter

    // Decide how to advance nextNode and ensure the target split center exists
    if nextNode.nexti.level != currentNode.level:
        nodeTargetSplitedCenter = nextNode.split()
        nextNode = nextNode.nexti
    else if nextNode.nextj.level != currentNode.level:
        nextNode = nextNode.nextj
    else:
        // If neither nexti nor nextj require a split, advance along nextk as fallback
        nextNode = nextNode.nextk

// After completing the nodeI loop, repeat the same process starting from the original start node's nodeJ
// and continue repeating until all target areas have next-level references (nexti, nextj, nextk are non-null).
```

### Notes & Implementation details

- split() on a Node returns the newly created center node (per Node.split() specification). Planet.split() must use those returned center nodes for rewiring.
- All level comparisons refer to the node.level (or split depth). Comparing levels allows the algorithm to detect whether a neighbour has already been processed to the same depth.
- Pointer assignments (nexti/nextj/nextk) must preserve reciprocity where required by the topology (i.e., when setting A.nexti = B, ensure the corresponding reciprocal pointer on B is set if the topology expects it).
- This specification assumes consistent use of child index naming (I, J, K) mapped to children[0..2] for implementation.

---
