# Grid navigation

`navigation::Grid` provides deterministic A* pathfinding over a bounded XZ
grid. Each cell stores walkability, elevation, and a positive traversal-cost
multiplier. World points are returned at cell centers using the grid origin,
cell spacing, and cell elevation. Use `cellAt` to map world-space X/Z
coordinates back to a cell.

For heightfield levels, `Grid::fromHeightfield` samples each terrain cell and
sets steep cells non-walkable from a maximum slope angle. It copies the sampled
elevations and spacing, so later terrain edits require rebuilding the grid or
updating the affected cell data explicitly.

```cpp
kiloria::navigation::Grid nav(128, 128, {-64.0, 0.0, -64.0});
nav.setWalkable({20, 20}, false);       // mark a solid obstacle
nav.setTraversalCost({21, 20}, 4.0);    // discourage a muddy tile
nav.setElevation({22, 20}, 0.25);      // a quarter-meter step

auto route = nav.findPath({2, 2}, {120, 120}, {
    .allowDiagonal = true,
    .simplify = true,
    .maximumStepHeight = 0.5,
    .maximumExpandedNodes = 100'000
});

// Or derive the grid dimensions, elevation, and slope mask from terrain:
auto terrainNav = kiloria::navigation::Grid::fromHeightfield(landscape, 0.78);
```

Diagonal moves cannot cut through the corner of blocked cells. Step height is
checked between adjacent cells; the simplified route also checks every crossed
cell and its elevation transition. A* uses physical cell spacing, average
endpoint traversal cost, and an admissible distance heuristic. The default
search cap bounds memory and frame time; no path is returned if the budget is
exhausted. `GridPath` reports world-space waypoints, accumulated grid cost,
expanded-node count, and the grid revision used for the search. Recompute a
route when its revision differs from `Grid::revision()` after edits.

Grid dimensions are capped at 8,192 per side and 4,194,304 cells in total.
This is a gameplay navigation grid, not a navmesh baker or a replacement for
collision queries. `kiloria_navigation` demonstrates obstacles, weighted
ground, and pathfinding without requiring a renderer or platform window.
