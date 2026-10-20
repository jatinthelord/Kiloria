# Input actions

Kiloria's `input::ActionMap` translates backend key, mouse, and gamepad events
into named gameplay actions. The engine does not depend on a particular window
library: the host forwards each event with `setKeyState`,
`setMouseButtonState`, `setGamepadButtonState`, `setGamepadAxisState`,
`addPointerDelta`, or `addScrollDelta`.

```cpp
kiloria::input::ActionMap input;
using namespace kiloria::input;
input.bindAdditional("jump", Binding{Control{ControlKind::Key, 32}});
input.bindAdditional("jump", Binding{Control{ControlKind::GamepadButton, 0}});
input.bindChord("save", 83, {17}); // host key codes: S plus Ctrl

input.beginFrame();                 // snapshot previous action state
input.setKeyState(32, true);        // events arrive from the window adapter
if (input.pressed("jump")) player.jump();
if (input.down("jump")) player.applyHeldJump();
if (input.released("jump")) player.endJump();
```

`bind` replaces an action's current-context mapping. Use `bindAdditional` for
multiple physical inputs, `bindChord` for modifier chords, or `rebind(context,
action, binding)` to replace a mapping in a named context. `setActiveContext`
switches between gameplay and editor bindings. Key/button inputs are digital;
gamepad axes support a deadzone and scale, with all action values summed and
clamped to [-1, 1]. Negative scales invert an axis. `state(action)` reports the
analog value and frame-stable down/pressed/released edges. Pointer and scroll
deltas reset at `beginFrame`; use `pointerDelta()` and `scrollDelta()` to read
their pixel/step values without clamping them to the action range.

Physical key codes belong to the host adapter, which can use GLFW, SDL, or a
platform API and keep those dependencies outside the portable engine core. The
`kiloria_input_actions` example shows an editor save chord and keyboard/gamepad
movement binding.
