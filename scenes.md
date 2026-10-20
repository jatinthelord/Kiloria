# Scene files and hierarchy

`scene::SceneArchive` reads and writes the binary `.kls` scene format. The
version-2 snapshot preserves entity IDs, UTF-8 name bytes, optional parent IDs,
local position, Euler metadata, quaternion orientation, and scale.

```cpp
kiloria::scene::Scene scene;
const auto platform = scene.createEntity("Platform");
const auto lamp = scene.createEntity("Lamp");
scene.setParent(lamp, platform);

kiloria::scene::SceneArchive::save("Scenes/Room.kls", scene);
auto loaded = kiloria::scene::SceneArchive::load("Scenes/Room.kls");
```

`encode` and `decode` provide in-memory round trips. Loading validates unique
non-zero IDs, finite transforms, non-zero quaternions, parent references, and
hierarchy cycles before replacing scene state. Archives use little-endian fixed
width values, a versioned header, a 256 MiB file limit, a one-million-entity
limit, 4 KiB name limits, and an FNV-1a corruption checksum. The checksum detects
accidental corruption; it does not authenticate files.

The overload without a codec registry serializes core scene transforms only.
`GameObject` dimensions, mesh/material assignments, physics settings, scripts,
and renderer resources need explicit codecs and stable asset references before
a scene can be a complete editor project. Version-1 transform snapshots remain
readable. `examples/scene_archive.cpp` demonstrates a hierarchy round trip.

## Components

`Scene` also owns a type-indexed component store. Components are stored densely
per type and looked up by stable entity ID:

```cpp
struct Health { float current; float maximum; };
scene.addComponent<Health>(player, Health{100.0F, 100.0F});
if (auto* health = scene.getComponent<Health>(player)) health->current -= 10.0F;
scene.removeComponent<Health>(player);
```

`eachComponent<T>` iterates dense values and passes `(EntityId, T&)`. Do not add
or remove that component type inside its iteration callback; queue structural
changes for after the iteration. Components must be nothrow movable so dense
swap-removal preserves index integrity. Destroying an entity removes all its
components. `replaceEntities` clears all components. Persist them by registering
an explicit portable schema with `ComponentCodecRegistry`. The game defines
stored fields and byte encoding; pointers, GPU handles, and transient caches
should be rebuilt after loading. Schema names are stable project identifiers,
never compiler type names. Increment the schema version when its layout changes.
Missing schemas and version mismatches fail explicitly, and component payloads
are covered by the archive checksum and size limits. The `kiloria_components`
example demonstrates a portable `Health` codec.
For objects managed by `GameWorld`, use `game.destroyObject(object.id())` so its
scene entity, component pools, and physics collider are removed together. The
object exposes `sceneEntityId()` when tools need to inspect or attach components.
Pass `--write-starter` to that executable to create `Scenes/StarterWorld.kls`,
the scene asset referenced by `examples/starter.kproj`.
