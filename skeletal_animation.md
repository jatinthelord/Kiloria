# Skeletons and CPU skinning

`animation::Skeleton` stores up to 1,024 uniquely named joints. Joint input is
topologically ordered: a parent must appear before its children. Each joint
has a local bind transform and can optionally map to an entity ID tracked by
the existing `Animator`. `bindPose()` produces the local pose vector directly;
`samplePose(animator)` samples mapped tracks and fills unmapped joints from the
bind pose.

`SkinnedMesh` pairs valid indexed triangles with one four-influence record per
vertex. Weights must be finite, non-negative, and have a positive sum; they are
normalized at construction. `Skeleton::skinInto` reuses caller-owned vertex
storage, while `skin` returns an allocating mesh copy. Both deform positions,
inverse-transpose normals, and orthogonalized tangents using linear blend
skinning.

```cpp
kiloria::animation::Skeleton skeleton({
    {"root", std::nullopt, rootBind, rootAnimationEntity},
    {"hand", std::uint16_t{0}, handBind, handAnimationEntity}
});
const auto localPose = skeleton.samplePose(animator);
skeleton.skinInto(mesh, localPose, reusableVertexBuffer);
```

Bind matrices are computed from the local hierarchy and inverted as affine
matrices. Singular or numerically unstable bind transforms and animated joint
poses are rejected. The mesh input is capped at ten million vertices. This is
CPU linear blend skinning for tools and backend-neutral integration; dual
quaternion blending, GPU skinning buffers, animation import, and rig retargeting
remain future modules. `kiloria_skeletal_animation` connects a sampled clip to a
two-joint skin without requiring a graphics context.
