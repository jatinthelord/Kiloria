# Kiloria project manifest, version 1

The `.kproj` file is the project index, not an archive. Asset bytes live at
project-relative paths and are loaded by later import/render modules.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<KiloriaProject version="1" name="StarterWorld">
  <Assets>
    <Asset id="1001" type="scene" path="Scenes/StarterWorld.kls" />
    <Asset id="1002" type="material" path="Materials/Default.kmat" />
  </Assets>
  <Scene asset="1001" />
</KiloriaProject>
```

`assets::ManifestCodec::parse` validates in-memory XML. `read` applies the same
schema to a file, `write` returns deterministic XML, and `writeFile` saves it.
`GameWorld::loadManifest` reads a manifest and registers its asset descriptors.
Asset IDs are non-zero and unique; a startup scene must refer to an asset whose
type is `scene`.

The accepted XML subset supports a declaration, comments, quoted attributes,
the five predefined XML entities, and whitespace-only element content. DTDs,
external and numeric entities, mixed content, unknown attributes, excess nesting,
more than 100,000 assets, and files over 4 MiB are rejected. This keeps parsing
predictable and avoids external entity expansion. Paths are validated as
relative before registration and checked against the project root when resolved.
