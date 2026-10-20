# OpenGL renderer

Kiloria's renderer targets OpenGL 3.3 core. It does not create a window, choose
a display adapter, or create a GL context. A host application (for example,
GLFW, SDL, or a platform-specific layer) creates and makes a context current,
then supplies a procedure resolver.

```cpp
#include <kiloria.h>

kiloria::render::OpenGLRenderer renderer;
renderer.initialize([&window](const char* name) -> void* {
    return window.getOpenGLProcAddress(name); // adapt the host's loader API
});

auto mesh = renderer.createMesh(cpuMesh);
auto texture = renderer.createTextureRGBA8(width, height, rgbaPixels);
auto material = renderer.createMaterial(materialData, texture);

const auto projection = kiloria::math::Mat4::perspective(
    1.0471975512, aspect, 0.1, 1000.0);
const auto view = kiloria::math::Mat4::lookAt(eye, target, {0.0, 1.0, 0.0});
const kiloria::render::DrawItem item{mesh, material, transform.matrix()};
renderer.beginFrame(width, height);
renderer.draw(std::span(&item, 1), view, projection, eye, sun);
renderer.endFrame();
window.swapBuffers(); // presentation belongs to the host

renderer.destroyMaterial(material);
renderer.destroyTexture(texture);
renderer.destroyMesh(mesh);
renderer.shutdown(); // while the GL context is still current
```

The Studio viewport renders into an RGBA16F target, then runs a half-resolution
bloom pyramid, fitted ACES, and sRGB output. Material shaders must write linear
radiance when the host sets `linearOutput`; tone mapping happens once, in the
composite pass. The sun uses three stable shadow cascades fitted by
`render::fitDirectionalCascades`, with rotated Poisson filtering and a blend
across cascade splits. Screen-space ambient occlusion darkens indirect light
only. `kiloria_quality_frame` checks the cascade fit, the ACES curve, the bloom
pyramid, and the split-sum BRDF coefficients without a GL context.

The OpenGL 3.3 forward shader evaluates a Cook-Torrance microfacet BRDF with
GGX normal distribution, Schlick Fresnel, and Smith masking-shadowing. It
supports one directional light, up to 32 inverse-square point lights, and up to
16 radius-limited spot lights. Local light attenuation includes a smooth finite
range window; spot cones use inner/outer radians. The renderer applies exposure,
the same fitted ACES curve as the studio composite, then display gamma. The light arrays are bounded
by shader constants to keep the GLSL 3.3 path portable.

```cpp
std::array<kiloria::render::RendererPointLight, 1> points{{
    {.position={2.0, 3.0, 1.0}, .color={1.0, 0.72, 0.42, 1.0},
     .intensity=900.0, .radius=18.0}
}};
std::array<kiloria::render::RendererSpotLight, 1> spots{{
    {.position={0.0, 5.0, 0.0}, .direction={0.0, -1.0, 0.0},
     .color={0.7, 0.82, 1.0, 1.0}, .intensity=1200.0, .radius=22.0,
     .innerConeRadians=0.25, .outerConeRadians=0.45}
}};
renderer.draw(items, view, projection, cameraPosition, sun, points, spots, 1.15);
```

Meshes remain indexed triangles with position, normal, and UV attributes.
Textures are uploaded as caller-provided RGBA8 pixels and use linear filtering
and repeat wrapping. This 3.3 backend still does not implement shadow maps,
environment-map IBL, normal maps, transparency sorting, or a separate HDR
framebuffer; the editor's viewport has its own renderer path, and the optional
OpenGL 4.6 renderer supplies an HDR target and additional material channels.

The resolver must return valid OpenGL function addresses for the current context.
If the host loader represents function pointers differently from `void*`, adapt
that conversion in the host callback. Call all renderer methods on the thread
where the context is current. Destroy GPU resources and call `shutdown()` before
destroying the context; the renderer destructor also releases remaining GL
objects and therefore requires a live context if `shutdown()` was omitted.

The host is responsible for buffer swapping, window events, resize notifications,
and context lifetime. `GameObject::applyTexture` currently records an asset
association; applications still connect imported asset bytes to renderer
texture/material handles themselves.

## Cameras, picking rays, and culling

`render::Camera` owns perspective or orthographic projection settings, a world
position, and a quaternion orientation. Set its viewport after window resize,
then pass `view()` and `projection()` into the renderer. Its `screenRay` method
converts top-left-origin pixel coordinates to a normalized world-space ray;
orthographic rays originate at different points and share a direction.

`camera.frustum()` extracts six normalized OpenGL clip planes from the current
view-projection matrix. Use `Frustum::intersects(Aabb)` or
`intersectsSphere(center, radius)` to cull drawables before building the draw
list. Boundary intersections are retained to avoid popping at frustum edges.
This is a CPU-side camera/culling utility; `OpenGLRenderer::draw` still accepts
the matrices and draw items directly. `kiloria_camera` demonstrates a center
screen ray and a frustum visibility query without creating a GL context.

## Building a scene draw queue

Attach a `render::MeshRendererComponent` to a scene entity with registered
renderer handles and, when available, object-space bounds. The extractor resolves
parented world transforms, transforms the eight bounds corners into a world AABB,
culls against the camera, and sorts visible opaque items by material/mesh for
better resource locality:

```cpp
kiloria::render::MeshRendererComponent drawable;
drawable.mesh = meshHandle;
drawable.material = materialHandle;
drawable.localBounds = meshBounds;
scene.addComponent<kiloria::render::MeshRendererComponent>(entity, drawable);
auto queue = kiloria::render::SceneRenderExtractor::build(scene, camera);
renderer.draw(queue.items, camera.view(), camera.projection(), camera.position(), sun);
```

Components without local bounds remain visible and skip frustum testing. Layer
masks allow editor, gameplay, and overlay views to select separate entities.
`RenderQueue::entities` stays index-aligned with its sorted `items`, and stats
report visibility, culling, and mesh/material transitions. The extractor is
CPU-only and cannot validate whether handles still exist in a specific renderer;
`OpenGLRenderer::draw` reports stale or unknown handles. Transparency sorting,
instancing, and automatic asset-to-GPU handle management remain future work.
