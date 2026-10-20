# Scene editor history

`editor::History` records reversible authoring edits by stable scene entity ID.
It currently supports full transform replacement, renaming, and parent changes.
An ordinary edit becomes a single undo step; call `beginTransaction` and
`commitTransaction` to group related edits, such as moving several objects as
one editor gesture. `cancelTransaction` reverts the edits made in the active
transaction.

```cpp
kiloria::editor::History history;
history.beginTransaction("Move selected objects");
for (const auto id : selection) {
    auto transform = scene.find(id)->transform;
    transform.position += dragDelta;
    history.setTransform(scene, id, transform);
}
history.commitTransaction();

history.undo(scene);
history.redo(scene);
```

History is bounded by transaction count, command count, and transaction-label
length through `HistoryLimits`. A new edit after undo discards the redo branch.
The history object is intentionally scoped to one `Scene`; it does not own the
scene, persist edits, or record component changes and entity creation/deletion.
If a referenced entity is removed outside this history, the affected undo or
redo returns `false` without advancing the cursor. Names and transforms are
validated before a recorded change is applied. `kiloria_editor_history`
demonstrates an atomic multi-operation transaction and its undo/redo cycle.
Clear the history when replacing a scene; entity IDs are stable only for the
lifetime of the scene state in which the history was recorded.
