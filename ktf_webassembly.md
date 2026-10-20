# KTF API and browser/WebAssembly demo

KTF is Kiloria's versioned C runtime API for tools and embedders. The same
`include/ktf.h` calls work in native code and in an Emscripten module; `KTFWorld`
owns a real `GameWorld`, so object creation, transforms, registered texture
metadata, and physics delegate to the engine implementation. C++ exceptions are
translated into `KTFStatus` values and `ktf_last_error()` diagnostics. Object
IDs are 64-bit; browser builds enable WebAssembly BigInt integration.

The browser sample under `examples/wasm/` creates a dynamic box from JavaScript,
advances Kiloria physics through `ktf_step`, and draws the resulting 3D position
on a canvas. It is a small API/demo bridge, not the full editor or the OpenGL
renderer. The browser canvas is only the sample's presentation layer.

## Build with Emscripten

Install the Emscripten SDK and configure with its CMake toolchain. From the
project root in an Emscripten-enabled shell:

```sh
emcmake cmake -S . -B build-wasm -DKILORIA_BUILD_KTF_WASM=ON -DKILORIA_BUILD_EXAMPLE=OFF
cmake --build build-wasm --target kiloria_ktf_demo
```

Serve `build-wasm/kiloria_ktf_demo.html` over HTTP (browsers generally block
WebAssembly loading from `file://`) and open it in a modern browser. The output
includes the HTML shell, generated JavaScript glue, and `.wasm` binary. The
browser executes the binary; WebAssembly text format (`.wat`) is optional for
inspection and can be generated with `wasm2wat kiloria_ktf_demo.wasm` when the
WABT tools are installed. WAT is not required by the browser or KTF API.

The native API example is `kiloria_ktf_api`. Build it with the regular CMake
build to exercise the same KTF world and physics calls without Emscripten. If
Emscripten is unavailable on the current machine, native compilation verifies
the shared KTF C API, but only an Emscripten build verifies browser linkage.
