# First object: one small step at a time

The public header is `<kiloria.h>`. Link the `Kiloria::Kiloria` CMake target.
The complete first example is `examples/one_object.cpp`.

## 1. Create a world

```cpp
#include <kiloria.h>
kiloria::GameWorld game("MyGame");
```

The world owns a scene, an asset registry rooted at `MyGame`, and the objects
created through its facade.

## 2. Give one object a dimension

```cpp
auto& block = game.createObject("StoneBlock", {2.0, 1.0, 2.0});
```

The three values are width, height, and depth in world units. All dimensions
must be positive and finite.

## 3. Move it locally

```cpp
block.move({0.0, 1.0, 0.0});
```

This changes the scene object's local position immediately. `setPosition` sets
an absolute position; `rotate(axis, radians)` composes a quaternion rotation.
Objects start with static AABB colliders matching their dimensions. Call
`block.enablePhysics(mass)` and advance `game.stepPhysics(seconds)` to make one
dynamic. Contacts use an enclosing axis-aligned box even after object rotation.

## 4. Attach a texture asset

```cpp
game.registerTexture(2001, "Textures/stone.ktx2");
block.applyTexture(2001);
```

The registry checks the asset type and keeps its path within the project root.
This records the texture association; image decoding and GPU upload are renderer
and importer work that has not been implemented yet.

## 5. Request a server move

```cpp
block.requestNetworkMove({3.0, 0.0, 0.0});
```

This emits a versioned, finite-checked move-intent payload through the configured
request sender. It does not change local position. A real transport adapter must
authenticate the session, send the request, and wait for server validation and
replication before applying authoritative state.

## Observe object changes

```cpp
const auto listener = game.events().subscribe<kiloria::ObjectMoved>(
    [](const kiloria::ObjectMoved& event) {
        // Update a tool panel, sound system, or gameplay observer.
    });
block.move({1.0, 0.0, 0.0});
game.events().unsubscribe(listener);
```

Events are synchronous. A listener can safely add or remove subscriptions while
an event is being delivered; callback exceptions propagate to the operation that
published the event.
