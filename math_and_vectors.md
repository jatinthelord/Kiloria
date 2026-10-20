# Kiloria vector algorithms and gradient descent

## VectorHandler

`<kiloria.h>` exposes `kiloria::math::VectorHandler`, a checked set of
double-precision 3D operations: `add`, `subtract`, `scale`, `lerp`, `cross`,
`magnitude`, `clampMagnitude`, and `reflect`. Inputs and computed results are
checked for finite values. Reflection requires a non-zero normal; lerp accepts
any finite interpolation parameter, including values outside `[0,1]`.

These operations are available to C++ gameplay modules through the public
`<kiloria.h>` engine API and the `kiloria::math` headers.

## GradientDescent

`kiloria::math::GradientDescent::minimize` supports scalar and Vec3 objectives
with caller-provided objective and derivative callbacks. It validates the
learning rate, tolerance, initial values, callback results, and an iteration
cap of one million. It reports the final point, objective value, iterations,
and whether the gradient met the requested tolerance. Gradient descent is
local optimization: learning rates that are too high can diverge or converge
slowly, and a converged result is not a guarantee of a global minimum.

The `kiloria_math_lab` example runs callback-driven scalar and vector
optimization and prints its convergence results. C++ gameplay modules can call
the same vector and optimizer APIs directly.

Build and run the native math demo:

```powershell
cmake --build build --config Debug --target kiloria_math_lab
.\build\kiloria_math_lab.exe
```
