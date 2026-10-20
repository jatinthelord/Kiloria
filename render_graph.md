# Render graph

Kiloria's backend-neutral render graph models a frame as named passes with
declared resource reads and writes. It compiles those declarations into a stable
schedule, rejects transient resources read before initialization, detects
dependency cycles, derives producer/consumer barrier masks, and allocates
compatible transient resources into reusable physical slots when their
lifetimes do not overlap.

```cpp
#include <kiloria.h>

using namespace kiloria::render;
RenderGraph graph;
auto depth = graph.createResource({.name="SunDepth", .format=GraphFormat::Depth32F,
                                   .width=2048, .height=2048});
auto hdr = graph.createResource({.name="SceneHDR", .format=GraphFormat::RGBA16F,
                                 .width=1280, .height=720});
auto backbuffer = graph.importResource({.name="Swapchain", .format=GraphFormat::SRGBA8,
                                        .width=1280, .height=720});

graph.addPass("Shadow map", [&](RenderGraph::PassBuilder& pass) {
    pass.write(depth, ResourceAccess::DepthAttachmentWrite);
});
graph.addPass("Forward PBR", [&](RenderGraph::PassBuilder& pass) {
    pass.read(depth); // implies shadow pass -> lighting pass
    pass.write(hdr, ResourceAccess::ColorAttachmentWrite);
});
graph.addPass("Present", [&](RenderGraph::PassBuilder& pass) {
    pass.read(hdr);
    pass.write(backbuffer, ResourceAccess::ColorAttachmentWrite);
});

const auto plan = graph.compile();
if (!plan.valid) { /* show plan.errors in the editor */ }
graph.execute(plan, [](Barrier transition) {
    // Translate graph barrier bits to glMemoryBarrier / Vulkan pipeline barriers.
});
```

Graph resource IDs are graph-local handles, not OpenGL names. A renderer backend
uses each `PassExecution::resources` binding to resolve a handle to its native
texture or buffer object. Imported resources represent externally owned targets
such as swapchains and are never aliased. Transient byte estimates are lower
bounds: they exclude backend allocation granularity, metadata, and driver
alignment. Resource aliasing is safe only when the backend honors the compiled
schedule and creates compatible native allocations for each physical slot.

The graph is deliberately not an OpenGL command recorder. The editor can keep
its existing OpenGL 3.3 path while a backend adapter translates the plan to GL
barriers, Vulkan synchronization2, or D3D12 resource transitions. A compiled
plan is tied to the graph revision; mutate the graph and compile again before
execution. Pass callbacks should record commands on the graphics thread and
should not mutate the graph while executing.

`kiloria_render_graph` is a CPU-only example containing shadow, GBuffer,
deferred lighting, bloom, and tone-map/present passes. Its callbacks demonstrate
ordering and barrier delivery; they do not issue actual GPU draws.
