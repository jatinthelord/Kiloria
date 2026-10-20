# Kiloria Studio

Build the `kiloria_editor` CMake target to run the GLFW, Dear ImGui, and OpenGL
editor. The initial workspace places the scene viewport in the center, the
World Outliner and Details/Renderer tabs on the right, and Content Browser,
Output Log, C++ Script Editor, Animator Debugger, and Science Lab tabs along the
bottom. Drag tabs to rearrange the workspace; **View > Reset Unreal-style
Workspace** restores the initial docking layout.

## Scene and assets

- **File > New Scene** starts an untitled scene; no project manifest is needed.
- Scene archives use `.kls`. The optional XML project index uses `.kproj`.
- Import Wavefront OBJ meshes from the File menu or the Content Browser.
- Use the World Outliner to select, hide, duplicate, and delete actors. The
  Details panel edits transforms, material parameters, and physics body modes.
- Use **W/E/R** to choose move, rotate, and scale tools. The viewport supports
  perspective orbit, zoom, lit/wireframe display, and GLSL shader reload.

## OpenGL renderer

The viewport uses GLSL 3.30 shaders from `assets/shaders` with embedded shader
fallbacks. The renderer includes Cook-Torrance GGX lighting, cascaded sun shadows,
procedural day/night sky, height fog, half-resolution SSAO, HDR bloom, ACES tone
mapping, color grading, and FXAA. The Renderer panel exposes the live pass
controls and reports shader compile/link errors in the Output Log.

## C++ gameplay modules

The editor only treats native C++20 modules as scripts. Open the **C++ Script
Editor**, edit `Scripts/Native/GameplayModule.cpp`, and choose **Build + Hot
Reload**. Kiloria invokes the `kiloria_cpp_gameplay` CMake target, copies up to
80 compiler-output lines into Output Log, and asks `ModuleLoader` to replace the
active module only after a successful build. The module loader stages a unique
shadow copy and keeps the previous module alive if validation or state restore
fails. Module state is bounded to 16 MiB.

Native C++ modules execute in the editor process with its full privileges. They
are not sandboxed: only build and load code you trust. The versioned C ABI keeps
STL objects, exceptions, and module-owned allocations from crossing the DLL
boundary; it does not isolate the code.

The File menu opens/saves `.kls` scenes and imports OBJ files. Material archives
use the bounded `.kmat` format. `.kproj` manifests are optional for opening the
editor and are not a scripting language.
