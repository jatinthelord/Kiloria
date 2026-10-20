# Mesh asset import

`assets::ObjImporter` converts bounded Wavefront OBJ text into renderer-ready
`render::MeshData` submeshes. It handles 1-based and negative indices, polygon
faces (ear-clipped for concave polygons), object/group names, UV coordinates,
explicit normals, smoothing groups, generated normals/tangents, `usemtl` names,
and `mtllib` dependency names. Each `ObjSubmesh` includes its local-space AABB
for a `MeshRendererComponent`:

```cpp
auto model = kiloria::assets::ObjImporter::load("Models/ruins.obj");
for (const auto& submesh : model.submeshes) {
    auto meshHandle = renderer.createMesh(submesh.mesh);
    // Resolve submesh.materialName through the project's material registry.
}
```

`ObjImportOptions` bounds source bytes, line bytes, attribute counts, face size,
submesh count, and per-submesh vertex/index counts. `ObjImportError::line()`
identifies malformed input. Negative indices resolve relative to attributes
already declared, matching OBJ rules. Degenerate faces and polygons that cannot
be triangulated are rejected. Smoothing-off faces receive flat generated
normals; named smoothing groups accumulate area-weighted normals.

The importer returns material-library paths as dependencies but does not parse
MTL files, load textures automatically, handle vertex-color extensions,
preserve all OBJ free-form geometry, or upload GPU resources. BMP and TGA image
bytes can be decoded separately through `assets::ImageDecoder`; see
`docs/images.md`. Use `assets::Registry::resolve` for project-root confinement
before loading an asset path. The `kiloria_obj_import` example parses an
in-memory quad without requiring external files.
