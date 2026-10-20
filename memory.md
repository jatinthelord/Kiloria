# Temporary memory and frame arenas

`core::FrameArena` reserves a fixed byte budget and serves aligned allocations
from it. It does not fall back to the general heap: a budget overrun throws
`std::bad_alloc`, making capacity mistakes visible instead of quietly growing
memory. `std::pmr` containers can use the arena directly.

```cpp
kiloria::core::FrameArena frameMemory(2 * 1024 * 1024);
{
    kiloria::core::FrameArena::Scope frame(frameMemory);
    std::pmr::vector<kiloria::math::Vec3> visiblePositions(&frameMemory);
    visiblePositions.reserve(4096);
    // Fill from the current render/simulation frame.
} // The scope rewinds temporary storage even when control exits early.
```

Marks support nested scratch phases. `rewind(mark)` rejects stale marks from an
earlier reset and marks beyond the current cursor. Statistics expose current
usage, high-water usage, allocation count, and generation so an engine loop can
log and tune budgets. The monotonic resource ignores individual deallocation;
destroy PMR containers before rewinding their storage. The arena is not
thread-safe: give worker threads separate arenas or synchronize access outside
the allocator. This is a scoped temporary-memory tool, not a replacement for
ownership of long-lived objects or a general leak detector.

Build and run `kiloria_frame_arena` for a 512-transform allocation example.
