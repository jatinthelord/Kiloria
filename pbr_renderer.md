# Kiloria OpenGL 4.6 PBR renderer

`kiloria::render::OpenGL46Renderer` is the opt-in 4.6 backend. The existing
`OpenGLRenderer` remains available for the Studio's current 3.3 context. This
class does not silently lower the shader version: the host must create a 4.6
core context and pass a valid procedure resolver while that context is current.

## Frame sequence

```cpp
kiloria::render::OpenGL46Renderer renderer;
renderer.initialize(resolveGlProc, {.width = 1600, .height = 900});

renderer.setDirectionalLights(directionalLights);
renderer.setPointLights(pointLights);
renderer.setSpotLights(spotLights);
renderer.setEnvironment({irradianceCube, prefilteredCube, brdfLut, 1.0, mipCount});

renderer.beginFrame({0.025, 0.03, 0.04, 1.0});
renderer.draw(drawItems, view, projection, cameraWorldPosition);
renderer.endFrame();
// The HDR result is available from renderer.hdrColorTexture().
```

Set up meshes, textures, and materials on the render thread after initialization.
The renderer uses immutable DSA storage and persistent coherent vertex mappings.
An update must fit the original vertex and index capacities. Recreate the mesh to
grow it. Mesh writes and draw submission must be serialized; coherent mapping
does not remove CPU/GPU ownership hazards.

Texture map slots follow `PbrTextureSlot`: base color, metallic, roughness,
normal, ambient occlusion, and emissive. Base color and emissive assets should
be marked sRGB when they contain display-encoded color. Numeric material maps
must stay linear. Missing material maps use white multipliers, except normal
maps, which use the tangent-space +Z neutral value.

The embedded fragment shader evaluates the Cook-Torrance microfacet BRDF with
GGX normal distribution, Smith visibility, and Schlick Fresnel. Directional,
finite-radius point, and cone spot lights use direct illumination. IBL combines
an irradiance cube, a roughness-prefiltered radiance cube, and a split-sum BRDF
integration LUT. These are borrowed texture object names supplied by the asset
pipeline and remain caller-owned. This renderer does not bake them from an HDR
panorama.

Scene color is written to an RGBA16F target with a depth24 attachment. The final
fullscreen pass can apply exposure and an ACES fitted curve, then performs
gamma encoding unless the output framebuffer is configured for sRGB conversion.
The HDR target is exposed for compositing into the Studio viewport.

The current host editor still creates an OpenGL 3.3 context and uses
`OpenGLRenderer`. `OpenGL46Renderer` is compiled into the library but is not yet
selected by the editor startup path. The editor must request a 4.6 context,
initialize this class, route its scene queue through the PBR handles, and sample
the HDR texture in the viewport before the running Studio will display this
pipeline. Shadows, transparency, bloom, temporal AA, light clustering, and IBL
baking remain follow-up renderer stages.
