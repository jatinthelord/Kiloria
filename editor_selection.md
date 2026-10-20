# Editor selection

`editor::Selection` keeps an ordered multi-selection using scene entity IDs.
It supports replacing, adding, toggling, and removing selected entities, as
well as inclusive range selection in scene order and selecting a whole entity
subtree. The primary entity is the latest explicit selection target; the
anchor is retained for range gestures. Each effective change increments the
selection revision so editor panels can avoid rebuilding unchanged views.

```cpp
kiloria::editor::Selection selection;
selection.select(scene, clickedEntity);
selection.add(scene, anotherEntity);                    // Ctrl-style add
selection.selectRange(scene, shiftTarget);              // scene-order range
selection.selectSubtree(scene, groupedModel, true);     // include descendants

if (const auto pivot = selection.worldPivot(scene)) {
    // Position a transform gizmo at the mean world-space entity origin.
}
```

Selection capacity is configurable with `SelectionLimits`. Mutating selection
methods reject missing entities or over-capacity results without partial
updates. `prune(scene)` removes entities deleted by other editor operations and
repairs primary/anchor state. The selection object does not own the scene and
should be pruned after scene replacement or deletion. `kiloria_editor_selection`
demonstrates subtree selection, multi-select, pivot calculation, and pruning.
