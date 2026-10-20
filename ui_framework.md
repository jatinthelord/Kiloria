# Kiloria retained editor UI (Step 1)

This subsystem establishes the editor's retained-mode UI architecture. It is
not a wrapper around Dear ImGui and it does not paint through Win32 controls.
It owns widget trees, measurement/arrangement, theme resolution, routed input,
drag/drop state, docking semantics, and a backend-neutral paint contract.

## Runtime flow

```text
OS input ──> platform adapter ──> EventRouter ──> capture -> target -> bubble
                                              ├── pointer capture / keyboard focus
                                              └── drag enter / over / leave / drop

DockManager ──> split tree + tab stacks ──> DockArea rectangles
                                          └── active panel widget tree

Widget tree ──> measure(available) ──> arrange(bounds) ──> PaintContext backend
     └──────── StyleSheet selectors ──> inherited ComputedStyle ────────────┘
```

The platform adapter is responsible for converting native events to logical
pixels and implementing `PaintContext` with OpenGL, Vulkan, or another backend.
This makes the core independent of Win32 and keeps DPI scaling, GPU batching,
font atlases, clipping, and platform window handles out of widget state.

## Core contracts

- `Widget` has one parent, stable runtime identity, cached desired size, arranged
  bounds, enabled/visibility/hit-test state, and three event phases.
- `Panel` owns children with `shared_ptr`; parent links are weak. Cycles and
  attaching a child to two parents are rejected.
- `BoxPanel` supports fixed slots and weighted flex slots. `CanvasPanel` supports
  normalized anchors, edge offsets, and alignment. `GridPanel` supports fixed,
  auto, and weighted-star tracks with spans.
- `StyleSheet` resolves type, class, and ID selectors by specificity then source
  order. Foreground and font size inherit; box/background properties do not.
- `EventRouter` routes preview/capture top-down, target, then bubble bottom-up.
  It keeps weak pointer-capture and focus references, so destroyed widgets are
  not retained by input state. Typed drag payloads are shared across enter/over/
  leave/drop notifications.
- `DockManager` stores panels by stable string ID in a split/tab tree. Panels can
  be tabbed, split before/after a target, activated, closed, and resized. Its
  `layout()` returns active stack geometry for the UI backend to paint.
- `FrameArena` is per-frame scratch storage. Reset invalidates all frame
  allocations; persistent widget state must never be stored in it.

All core UI operations are intended for the editor/UI thread. The frame arena is
not thread-safe. A platform adapter should enqueue worker-thread completions and
apply them at a frame boundary. Event handlers must not retain borrowed `Event`
references or frame-arena pointers.

## Controls, docking widgets, reflection, and backend boundary

`controls.hpp` adds `OverlayPanel`, `SSplitter`, `STextBlock`, `SImage`,
`SBorder`, `SButton`, and `SCheckBox`. Box panels accept `SBoxSlot` values with
padding, auto/fill rules, weights, and cross-axis alignment. Image brushes carry
a texture handle, source dimensions, tint, and nine-slice insets; `SBorder`
emits nine destination quads through `PaintContext::drawImage`.

`SDockingArea` is the widget integration layer over `DockManager`. Register a
panel with a stable ID and content widget, then use `tabPanel`, `splitPanel`,
`activate`, and `closePanel`. `SDockTab` handles pointer activation. The widget
adapter paints tab strips and arranges active panel content. The docking model
remains the source of truth for split geometry and tab state.

`reflection.hpp` provides `FProperty` snapshots and writes, type metadata, and
`SDetailsPanel`. Types opt in explicitly with `KILORIA_REFLECT_BEGIN`,
`KILORIA_REFLECT_PROPERTY[_META]`, and `KILORIA_REFLECT_END`; the detail panel
creates editable rows for scalar, string, vector, quaternion, and color values.
Reflection is opt-in: C++ has no safe general-purpose runtime field discovery.
Custom enum labels or nested-struct editors should be registered explicitly.

`renderer.hpp` defines `FSlateRenderer`, `UiDrawList`, and `FSlateGpuDevice`.
The renderer owns frame aggregation, adjacent-command coalescing, GLSL 4.60
shader sources, and one vertex upload per frame. A context-bound platform adapter
must implement device calls for VAO/VBO creation, streaming upload, scissor,
texture binding, blending, and draw submission. This repository does not yet
contain that GL 4.6 device implementation, a font atlas, or a native
`IWindowPlatform` implementation. `SWindow` and `SViewport` provide lifecycle
and widget interfaces; they do not manufacture OS or GPU resources.

Example reflection registration:

```cpp
struct FTransform {
    kiloria::math::Vec3 location{};
    kiloria::math::Quaternion rotation{};
    kiloria::math::Vec3 scale{1.0, 1.0, 1.0};
};
KILORIA_REFLECT_BEGIN(FTransform)
    KILORIA_REFLECT_PROPERTY_META(location, "Transform", "World-space position")
    KILORIA_REFLECT_PROPERTY_META(rotation, "Transform", "Unit quaternion orientation")
    KILORIA_REFLECT_PROPERTY_META(scale, "Transform", "Local scale")
KILORIA_REFLECT_END()

auto transform = std::make_shared<FTransform>();
auto details = std::make_shared<kiloria::ui::SDetailsPanel>();
details->inspect(transform);
```

## Workspace example

```cpp
#include <kiloria.h>

using namespace kiloria::ui;

DockManager docks;
docks.registerPanel({"Viewport", "Scene View", std::make_shared<Panel>(), false});
docks.registerPanel({"Outliner", "Outliner", std::make_shared<Panel>(), true});
docks.registerPanel({"Details", "Details", std::make_shared<Panel>(), true});
docks.registerPanel({"Content", "Content Browser", std::make_shared<Panel>(), true});
docks.dockSplit("Outliner", "Viewport", DockAxis::Horizontal, false, 0.74);
docks.dockSplit("Details", "Outliner", DockAxis::Vertical, false, 0.52);
docks.dockSplit("Content", "Viewport", DockAxis::Vertical, false, 0.78);

for (const DockArea& area : docks.layout({0, 0, 1600, 900})) {
    const DockPanel* active = docks.panel(area.activePanel);
    if (!active) continue;
    active->content->measure({area.bounds.width, area.bounds.height});
    active->content->arrange(area.bounds);
}

StyleSheet theme;
theme.addRule({.type = "Panel", .classes = {"dock-surface"}},
              {.background = Color{0.055F, 0.065F, 0.080F, 1.0F},
               .borderColor = Color{0.16F, 0.19F, 0.23F, 1.0F},
               .borderWidth = 1.0});
docks.panel("Viewport")->content->addStyleClass("dock-surface");
theme.applyTree(docks.panel("Viewport")->content);
```

The paint backend calls `root->paint(context)` after layout. Custom controls
override `onPaint` and the event hooks; they do not create their own native child
windows. To bridge this framework into the current Win32 editor, the next
integration unit is an `HWND` input adapter plus an OpenGL `PaintContext`, then
panel widgets for Scene View, Outliner, Details, and Content Browser.

## Implementation phases after this foundation

1. Bind a platform adapter and OpenGL UI painter; replace the current native
   control layout with the retained Kiloria widget tree.
2. Add tab-strip/splitter rendering and pointer-driven dock previews on top of
   `DockManager` operations.
3. Add reflection metadata and property editors, then bind Details and Outliner
   to the scene model with undoable commands.
4. Add virtualized content-browser rows, thumbnail cache, and asset drag payloads.
5. Add a frame profiler and test input, layout, dock mutation, and DPI behavior.

This is Step 1's interaction/layout core. It does not claim that the existing
editor is already visually identical to Unreal; the native editor integration
and the custom paint backend are still required for that result.
