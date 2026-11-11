# Linear Regression

`sciencekit.linear_regression()` builds an ordinary least-squares (`f64`) regression estimator backed by the `sciencekit_ffi` C ABI.

## Constructing and configuring the builder

Call `sciencekit.linear_regression()` to allocate an opaque builder userdata. Configuration setters mutate the builder in place and return the **same** userdata (`builder == builder:include_intercept(true)`), so calls chain cleanly:

```lua
local sciencekit = require("sciencekit")

local builder = sciencekit.linear_regression()
  :include_intercept(true)
  :backend("faer")

assert(builder.family == "linear-regression")
assert(builder.dtype == "f64")
assert(not builder.closed)
```

### Configuration methods

| Method | Accepted values | Behavior |
| --- | --- | --- |
| `:include_intercept(enabled)` | `boolean` | Controls whether an intercept (bias) term is solved alongside the feature weights. |
| `:backend(backend)` | `"faer"` (`0`), `"matrix-multiply"` / `"matrixmultiply"` (`1`), `"ndarray-linalg"` / `"ndarray_linalg"` (`2`) | Selects the underlying `sciencekit` math backend. Passing an unknown string, out-of-range integer, or any other type raises `sciencekit invalid-argument (3)` before invoking the C ABI. |

<div class="sk-box sk-box--warning">
  <p><strong>Family-specific setters:</strong> Calling a scaler-only setter such as <code>builder:with_mean(true)</code> or <code>builder:with_std(true)</code> on a linear-regression builder raises <code>sciencekit unsupported (4)</code>.</p>
</div>

## Fitting, predicting, and scoring

`builder:fit(features, targets)` validates and copies both tables into contiguous `f64` buffers, fits the model via `sk_ffi_fit`, and returns a new fitted-model userdata (`model.family == "linear-regression"`). The builder itself stays open and reusable until closed or garbage-collected.

```lua
local sciencekit = require("sciencekit")

local features = {
  { 0.0, 0.0 },
  { 1.0, 0.0 },
  { 0.0, 1.0 },
  { 2.0, 3.0 },
}
local targets = { 1.0, 3.0, 4.0, 14.0 }

local model <close> = sciencekit.linear_regression()
  :include_intercept(true)
  :backend("faer")
  :fit(features, targets)

local predictions = model:predict(features)
assert(#predictions == #targets)
assert(#predictions[1] == 1)

for index = 1, #targets do
  assert(math.abs(predictions[index][1] - targets[index]) <= 1e-8)
end

local r_squared = model:score(features, targets)
assert(math.abs(r_squared - 1.0) <= 1e-8)
```

### Target table formats

Both `builder:fit(features, targets)` and `model:score(features, targets)` accept targets in two shapes, provided `#targets == #features`:

1. **Flat sequence of numbers:** `{ 2.0, 4.0, 6.0 }`
2. **Column of one-element rows:** `{ { 2.0 }, { 4.0 }, { 6.0 } }`

```lua
local sciencekit = require("sciencekit")

local features = { { 1.0 }, { 2.0 }, { 3.0 } }
local column_targets = { { 2.0 }, { 4.0 }, { 6.0 } }

local model <close> = sciencekit.linear_regression():fit(features, column_targets)
assert(math.abs(model:score(features, column_targets) - 1.0) < 1e-8)
```

Omitting `targets` on linear regression (`sciencekit.linear_regression():fit(features)`) raises `sciencekit invalid-argument (3): linear regression requires targets`.

### Prediction output shape

`model:predict(features)` **always returns a table of one-element rows** (`{ { y1 }, { y2 }, ... }`), matching the uniform matrix output convention used across the binding:

- `#predictions` equals the number of input sample rows.
- `#predictions[row]` equals `1`.
- `predictions[row][1]` is the predicted `number` (`f64`) for `features[row]`.

## Scaling to larger tables

From `src/large_tests.rs`, fitting and predicting on a `2000 × 20` table in Lua exercises the full row-major copy and ABI round trip within double-precision tolerance:

```lua
local sciencekit = require("sciencekit")

local rows, columns = 2000, 20
local features, targets = {}, {}
for row = 1, rows do
  local values = {}
  for column = 1, columns do
    values[column] = ((row * column) % 97) * 0.25 + column
  end
  features[row] = values
  targets[row] = 2.5 * values[1] - 1.5 * values[2] + 0.25
end

local model <close> = sciencekit.linear_regression():fit(features, targets)
local predictions = model:predict(features)
assert(#predictions == rows)

local worst = 0.0
for row = 1, rows do
  local delta = math.abs(predictions[row][1] - targets[row])
  if delta > worst then worst = delta end
end
assert(worst < 1e-8)
```

## Unsupported operations on linear regression

Calling `model:transform(features)` on a fitted `linear-regression` model raises `sciencekit unsupported (4)`. Conversely, `model:save(path)` is supported on fitted `linear-regression` models — see [Model Persistence](./persistence.md).
