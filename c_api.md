# C and C++ game-author API

The public header is `#include <kiloria.h>`. A C++ translation unit receives the
full C++20 API. A C translation unit receives the stable opaque-handle interface,
so C code never has to know the layout of the engine's C++ classes or objects.

```c
#include <kiloria.h>

KiloriaWorldHandle* world = NULL;
KiloriaStatus status = kiloria_world_create("MyGame", &world);
uint64_t crate = 0;
const KiloriaDimension box = {1.0, 1.0, 1.0};
if (status == KILORIA_STATUS_OK)
    status = kiloria_world_create_object(world, "Crate", box, &crate);
if (status == KILORIA_STATUS_OK)
    status = kiloria_object_move(world, crate, 0.0, 1.0, 0.0);
kiloria_world_destroy(world);
```

Every fallible call returns a `KiloriaStatus`; C++ exceptions are caught at the
boundary. `kiloria_last_error` copies a thread-local diagnostic. Texture paths
must be relative to the project root and are checked by the existing asset
registry. The complete C quickstart registers an asset, creates and transforms
an object, applies the texture, and steps physics; build target
`kiloria_c_quickstart` compiles the source as C.

Network movement stays an explicit server request. A C host can install a
`KiloriaRequestSendFn`, then call `kiloria_object_request_network_move`. The
callback receives a server-generated request ID, route name, and borrowed payload
bytes. Copy the payload before asynchronous use, keep `user_data` alive until the
callback is cleared or the world is destroyed, and return nonzero only after a
real authenticated transport accepts it. The engine does not infer trust from a
client-supplied identity or treat local state changes as server authority.

## Rust movement guard

GNU-compatible builds use the small `no_std` crate in `rust/guard`. CMake builds
it through Cargo and links its displacement validator into the authenticated
object-move route. It rejects non-finite values, arithmetic overflow, and moves
outside the application-configured distance envelope. On toolchains without
Cargo or with an incompatible C++ linker, the equivalent C++ validation path is
used. The route still requires an authenticated transport identity and per-object
authorization; the Rust check is a numerical input guard, not authentication,
encryption, or full server-side physics. The crate has no third-party
dependencies.
