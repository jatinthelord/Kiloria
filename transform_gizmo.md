# Editor transform gizmos

`editor::TransformGizmo` applies world-space translate, rotate, and positive
uniform-scale gestures to a `Selection`. Each gesture is recorded as one
`editor::History` transaction, so a drag can be undone in one step. Rotation
and scaling use the selection's mean world-space pivot by default; pass an
explicit finite pivot to use a custom gizmo center.

```cpp
kiloria::editor::TransformGizmo::translate(scene, selection, history, {0, 1, 0});
kiloria::editor::TransformGizmo::rotate(scene, selection, history,
    {0, 1, 0}, 0.25); // radians
kiloria::editor::TransformGizmo::scale(scene, selection, history, 1.1);
```

When both a hierarchy parent and its descendant are selected, the gesture is
applied to the highest selected ancestor only; descendants inherit the same
group transform, avoiding double movement. Separate selected roots retain
their relative world placement. Children beneath unselected parents are
converted back to local space using the parent's world position, orientation,
and scale. As with `scene::Transform`, this decomposition cannot represent
shear from rotated non-uniformly scaled ancestors; a zero-scale parent cannot
be edited in world space. Gestures validate finite inputs and group all local
transform writes into a single undoable command. `kiloria_transform_gizmo`
demonstrates group translation, rotation, scale, and undo.
