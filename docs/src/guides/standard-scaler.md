# Standard Scaler

`sciencekit.standard_scaler()` builds a column-wise feature standardizer (`family == "standard-scaler"`, `dtype == "f64"`) over the `sciencekit_ffi` C ABI.

## Constructing and configuring the scaler

```lua
local sciencekit = require("sciencekit")

local builder = sciencekit.standard_scaler()
  :with_mean(true)
  :with_std(true)
  :backend("faer")

assert(builder.family == "standard-scaler")
assert(builder.dtype == "f64")
```

### Configuration methods

| Method | Accepted values | Behavior |
| --- | --- | --- |
| `:with_mean(enabled)` | `boolean` | When `true`, subtracts each column's learned mean during `transform`. |
| `:with_std(enabled)` | `boolean` | When `true`, divides each column by its learned standard deviation during `transform`. |
| `:backend(backend)` | `"faer"` (`0`), `"matrix-multiply"` / `"matrixmultiply"` (`1`), `"ndarray-linalg"` / `"ndarray_linalg"` (`2`) | Sets the configured math backend transferred to the fitted model. |

Calling `builder:include_intercept(...)` on a `standard-scaler` builder raises `sciencekit unsupported (4)`.

## Fitting and transforming matrices

`standard_scaler` is unsupervised: call `builder:fit(features)` with a single rectangular matrix table, then call `model:transform(features)` to obtain a fresh table with the exact same row and column dimensions:

```lua
local sciencekit = require("sciencekit")

local features = {
  { 0.0, 0.0 },
  { 2.0, 2.0 },
  { 4.0, 4.0 },
  { 6.0, 6.0 },
}

local model <close> = sciencekit.standard_scaler()
  :with_mean(true)
  :with_std(true)
  :fit(features)

local transformed = model:transform(features)
assert(#transformed == 4)       -- one row per sample
assert(#transformed[1] == 2)    -- one column per feature

-- Symmetric first and fourth rows sum to 0.0 after centering:
assert(math.abs(transformed[1][1] + transformed[4][1]) <= 1e-8)
-- Identical input columns share the same scaled values:
assert(math.abs(transformed[1][1] - transformed[1][2]) <= 1e-8)
```

## Centering without scaling (`with_std(false)`)

Disabling `:with_std(false)` subtracts each column's mean (here `3.0`) while leaving unit steps unchanged (from `tests/run.lua`):

```lua
local sciencekit = require("sciencekit")

local features = {
  { 0.0, 0.0 },
  { 2.0, 2.0 },
  { 4.0, 4.0 },
  { 6.0, 6.0 },
}

local centered <close> = sciencekit.standard_scaler()
  :with_mean(true)
  :with_std(false)
  :fit(features)

local offset = centered:transform(features)
assert(math.abs(offset[1][1] - (-3.0)) <= 1e-8)
assert(math.abs(offset[4][1] - 3.0) <= 1e-8)
```

## Unsupported operations on standard scaler

Fitted `standard-scaler` models implement `model:transform(features)` and `model:close()`. Calling regressor-specific methods raises `sciencekit unsupported (4)` (or `io` / `unsupported` from the C ABI):

```lua
local sciencekit = require("sciencekit")

local scaler <close> = sciencekit.standard_scaler():fit({ { 1.0 }, { 2.0 } })
local ok, err = pcall(function()
  scaler:predict({ { 1.0 } })
end)
assert(not ok)
assert(tostring(err):find("sciencekit unsupported (4)", 1, true))
```
