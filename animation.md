# Animation clips and blending

`TransformTrack` samples position, Euler metadata, scale, and normalized
quaternion orientation. `AnimationClip` groups tracks by scene entity ID and
stores sorted gameplay markers. `Animator` advances multiple named clips,
supports clamp, loop, and ping-pong playback, and returns markers crossed by
each update, including loop wraparound and reverse playback.

```cpp
kiloria::animation::AnimationClip walk("walk");
walk.setTrack(characterRoot, kiloria::animation::TransformTrack({
    {0.0, standingPose}, {0.45, leftStep}, {0.9, standingPose}}));
walk.addMarker({0.45, "footstep", "left"});

kiloria::animation::Animator animator;
animator.setBindPose(characterRoot, standingPose);
animator.addClip(std::move(walk));
const auto locomotion = animator.addLayer({
    .clip = "walk", .speed = 1.0, .weight = 0.8F,
    .playback = kiloria::animation::Playback::Loop});
const auto events = animator.update(deltaSeconds);
const auto pose = animator.sample(characterRoot);
```

Override layers blend position, scale, Euler metadata, and orientation by
weight. The unused portion of a total override weight below one comes from the
bind pose; weights above one are normalized. Additive layers apply translation,
scale, and Euler deltas relative to the track's first pose, plus a local
quaternion delta. Layer order is deterministic and additive layers apply after
override blending. `update` rejects non-finite/negative frame deltas and limits
marker traversal to 4,096 events and clip cycles per update to keep large time
jumps bounded.

The animator outputs transforms; a host/editor applies them to scene entities
or a renderer pose buffer. Skeletons can map joint names to animator entity IDs
and use those sampled local transforms for CPU skinning; see
`docs/skeletal_animation.md` and `kiloria_skeletal_animation`. Animation-file
import, GPU skinning buffers, and root-motion scheduling into physics remain
future systems. `kiloria_animation` demonstrates two weighted clips and a
footstep marker.
