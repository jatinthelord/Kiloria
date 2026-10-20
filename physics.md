# Kiloria physics and collision

Kiloria has separate simulation paths:

- `physics::World` integrates sphere bodies with accumulated forces, gravity,
  sphere contacts, a ground plane, and buoyancy/drag against fluid surfaces.
- `physics::CollisionWorld` integrates axis-aligned box colliders, detects box
  contacts with sweep-and-prune, and resolves dynamic contacts with sequential
  normal and friction impulses.
- `terrain::Heightfield` provides sampled ground support to the kinematic
  character controller; it is not yet integrated into dynamic body contacts.

Both use meters, seconds, kilograms, and Y-up coordinates. Advance them with a
fixed timestep; `core::FixedStepClock` bounds catch-up after a long frame.
`core::Runtime::advanceFrame` steps the sphere world and the box world on that
same clock.

## Beginner objects

`GameWorld::createObject` creates a static box collider whose full size comes
from `Dimension`. Make it dynamic and advance the world like this:

```cpp
auto& crate = game.createObject("Crate", {1.0, 1.0, 1.0});
crate.enablePhysics(2.0, 0.15, 0.6); // mass, restitution, friction
crate.setVelocity({1.0, 4.0, 0.0});
game.stepPhysics(1.0 / 60.0);
```

`stepPhysics` syncs resolved collider positions into scene transforms and emits
`ObjectMoved` events. `GameWorld::collisionWorld()` exposes contacts and advanced
filter/trigger controls. A collider with mass zero is static. A trigger reports
overlaps but does not push bodies apart. Category and mask bits must match in
both directions for a pair to collide.

`CharacterController` performs a kinematic sweep using an upright AABB proxy.
It adds acceleration-based ground/air movement, jump buffering, coyote time,
ground probes, ceiling/wall state, and low-step traversal. It queries the
collision world without registering a dynamic collider; call its `step` on the
same fixed update cadence as the world. Without a heightfield attached, it does
not solve slopes or moving-platform carry. The current proxy has square corners
and is not a capsule controller.

```cpp
kiloria::physics::CharacterController player(world, {0.0, 1.0, 0.0});
player.setMoveInput({1.0, 0.0, 0.0}); // direction is normalized and constrained to XZ
if (jumpButtonPressed) player.requestJump();
player.step(1.0 / 60.0);
const auto& state = player.state(); // center, velocity, grounded, wall/ceiling
```

Call `player.setTerrain(&heightfield)` to make the controller sample and follow
a regular-grid terrain surface. The terrain is non-owning and must outlive the
controller. `CharacterConfig::maximumSlopeAngleRadians` and `stepHeight` bound
uphill traversal; terrain sampling uses the character center rather than a full
capsule footprint. Dynamic rigid-body contact with heightfields is not part of
the current solver.

The default step accepts at most 0.25 seconds per call to preserve stable
movement response. Split longer frame times with `FixedStepClock`. The
`moveKinematicAabb` query is also available for editor gizmos, moving platforms,
and other non-dynamic actors; triggers are ignored by the blocking sweep.

## Current solver boundaries

`physics::World` fluid surfaces are infinite horizontal planes. Sphere
submersion uses the analytic spherical-cap volume fraction
`h²(3r-h)/(4r³)`, where `h` is the submerged cap height and `r` is sphere
radius. Buoyancy follows Archimedes' relation `F_b = ρ g V_sub` opposite to
gravity. Linear and quadratic relative-flow drag are integrated with implicit
velocity damping to reduce timestep sensitivity. Each body samples the most
submerged fluid surface, so overlapping planes do not stack buoyancy. This is a
useful buoyancy model, not a fluid solver: there is no wave simulation, water
displacement, surface tension, or spatially bounded fluid volume. The `Body`
reports the selected fluid ID, submerged fraction, and last-step force estimates.

`CollisionWorld::step` chooses a bounded number of internal substeps from the
fastest dynamic body, gravity, and the smallest collider half-extent. This
reduces tunneling for fast axis-aligned boxes without requiring callers to
change their frame loop; inspect `lastStepSubsteps()` and tune the cap with
`setMaximumAdaptiveSubsteps()` (default 32, supported range 1..256). This is a
discrete collision safeguard rather than exact continuous time-of-impact
solving; objects can still pass through thin geometry once the cap is reached.

The box solver is intentionally an axis-aligned approximation. Rotating a game
object changes its box to the conservative AABB that encloses the rotated shape;
it does not run oriented-box SAT. It has no angular velocity, joints, CCD,
fluid solver, or multithreaded broadphase yet. A dynamic box sleeps once it is
supported and slower than 0.06 m/s, and a hard contact wakes it. `wake` also
runs when gameplay code teleports the box or assigns a velocity.
High-speed objects can still tunnel through thin colliders if the adaptive
substep cap is reached. Use fixed outer timesteps and keep gameplay objects
inside a sensible numeric world range.

The broadphase sorts boxes by their minimum X bound, then checks Y and Z overlap
for candidates. Collision filters are applied before contact generation. The
impulse solver uses inverse mass, a small positional slop, configurable iteration
count, restitution, and geometric-mean friction. This is a useful engine
foundation, not a validated real-world engineering or chemistry simulator.

`examples/collision_world.cpp` demonstrates the standalone box world.
