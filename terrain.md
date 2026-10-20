# Editable heightfield terrain

`terrain::Heightfield` stores a bounded, regular Y-up grid of local height
offsets. `origin` is the world position of sample (0, 0), while `GridSpacing`
sets X/Z distance between samples. The initial implementation caps storage at
4,194,304 samples and each dimension at 8,192 to keep edit, query, and mesh
operations within explicit bounds.

```cpp
kiloria::terrain::Heightfield terrain(
    samplesX, samplesZ, {-128.0, 0.0, -128.0}, {1.0, 1.0}, std::move(heights));

if (auto surface = terrain.surfaceAt(playerX, playerZ)) {
    // surface->height and surface->normal can drive placement and ground queries.
}
const auto chunks = terrain.buildMeshes({.cellsPerChunk = 64, .uvScale = 0.1});
for (const auto& chunk : chunks) {
    const auto handle = renderer.createMesh(chunk.mesh);
    // Keep the handle with this terrain chunk for later drawing or rebuilding.
}
```

`heightAt` performs piecewise-linear world-space height lookup using the same
cell diagonal as the generated mesh. `surfaceAt` also returns a
finite-difference normal. `raycast` walks grid cells front-to-back and tests
the same two triangles emitted by the mesh builder, returning a ray parameter,
hit point, triangle normal, and cell coordinate. The ray direction need not be
normalized, so `parameter` is not necessarily a distance in meters.

`applyBrush` supports smooth-falloff raise/lower, flatten-to-world-height, and
neighbor-averaged smoothing. It returns exact `{x, z, before, after}` edits;
`applyEdits(edits, true)` undoes them and `applyEdits(edits, false)` reapplies
them. Brush changes are validated before mutation, which lets editor undo
history store compact sample edits instead of copying the whole terrain.

`buildMeshes` splits the grid into independently uploadable indexed chunks.
Neighboring chunks share boundary positions and normals, and each chunk carries
its grid origin and world AABB for culling. Regenerate chunks touched by terrain
edits and replace their GPU mesh handles on the render thread. This heightfield
can also ground a `physics::CharacterController` through its non-owning
`setTerrain` reference. The controller follows sampled elevations, respects a
maximum walkable slope and step height, and still collides with regular box
colliders. The heightfield must outlive every controller attached to it. This
is kinematic character support, not rigid-body heightfield contact. Tiled
streaming, material layers, erosion, and LOD selection remain future systems.
`kiloria_terrain` demonstrates sampling, ray picking, reversible sculpting,
chunk generation, and a character walking across the heightfield without an
OpenGL context.
