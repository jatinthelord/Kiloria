# C++ module ABI and hot reload

Kiloria scripts are native C++20 shared modules. The versioned, C-compatible
descriptor is in `include/kiloria/script/module_abi.h`; the transactional host is
`kiloria::script::ModuleLoader` in `include/kiloria/script/module_loader.hpp`.
The C ABI is used only to keep compiler-specific C++ layouts and allocation
ownership out of the boundary. It does not make a loaded module sandboxed.

## Lifecycle

```text
source -> CMake module build -> unique shadow copy -> ABI validation
        -> create/restore -> tick -> save state -> destroy -> unload
```

The active module remains loaded unless the candidate exports the expected entry
point, has a compatible descriptor, and successfully creates an instance using
the previous state. A failed build or load leaves the active module in place.
Reload and tick are serialized by the loader.

State is capped at 16 MiB. The host allocator owns bytes returned from
`save_state`; the host copies and releases them. `create` receives a borrowed span
that is valid only for that callback. A module must validate every host-provided
frame and state buffer and must not throw across the ABI.

## Trust boundary

Native C++ code executes inside the Kiloria process with its operating-system
permissions. The module ABI does not isolate memory, file access, or system
calls. Only load modules from trusted sources. A future untrusted-script feature
would require a separate process and a constrained authenticated protocol.

See `Scripts/Native/GameplayModule.cpp` for the example module and
`docs/scripting.md` for the editor workflow.
