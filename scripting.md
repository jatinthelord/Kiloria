# C++ gameplay modules

Kiloria's gameplay scripting workflow is C++20. A module is a CMake `MODULE`
target that exports `kiloria_script_module_get_api_v1` from
`include/kiloria/script/module_abi.h`. The editor's C++ Script Editor builds the
`kiloria_cpp_gameplay` target and hot-reloads the resulting shared library.

The stable C ABI is intentionally narrow. It passes fixed-width integers,
floating-point frame data, byte spans, and callbacks. No `std::string`, STL
container, C++ exception, or object with compiler-specific layout crosses the
module boundary. Each module owns its instance and destroys it through its own
callback.

Modules run in the host process with native operating-system privileges. The ABI
is not a sandbox and does not make untrusted code safe. Restrict module loading
to trusted source, and use a separate process plus an authenticated protocol if
untrusted gameplay code must be supported later.

## Authoring and reload

1. Edit `Scripts/Native/GameplayModule.cpp` or add another CMake module target.
2. Build the module with C++20 and the Kiloria include directory.
3. Export a static `KiloriaScriptModuleApiV1` descriptor and implement create,
   tick, save-state, and destroy callbacks.
4. In Studio, press **Build + Hot Reload**. Compiler diagnostics appear in the
   Output Log. Failed compilation leaves the active library unchanged.
5. During play, the engine calls `tick` with a finite frame delta clamped to the
   module ABI contract.

The loader copies a candidate library to a unique temporary path before loading
it. This allows the build system to replace the original file while a previous
version is active. State serialization uses the host allocator and is limited to
16 MiB. The next module receives borrowed state bytes during `create` and must
copy anything it needs to retain.

## Minimal module shape

```cpp
#include <kiloria/script/module_abi.h>

struct GameplayState { double elapsed{}; };

// Export the API using KILORIA_SCRIPT_EXPORT and the exact ABI entry-point name.
// Each callback must validate inputs and return a KiloriaScriptStatusV1 value.
```

The complete working module, including bounded state save/restore and logging,
is `Scripts/Native/GameplayModule.cpp`.
