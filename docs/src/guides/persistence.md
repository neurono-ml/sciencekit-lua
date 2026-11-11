# Model Persistence

Fitted linear-regression models can be serialized to disk with `model:save(path)` and restored into any Lua 5.4 VM with `sciencekit.load(path)`. Both entry points delegate directly to `sk_ffi_model_save` and `sk_ffi_model_load` in `sciencekit_ffi`, so files written by Lua are binary-compatible with C and Go consumers using the same ABI version.

## Saving and loading a fitted model

```lua
local sciencekit = require("sciencekit")

local features = {
  { 0.0, 0.0 },
  { 1.0, 0.0 },
  { 0.0, 1.0 },
  { 2.0, 3.0 },
}
local targets = { 1.0, 3.0, 4.0, 14.0 }
local path = os.tmpname()

do
  local model <close> = sciencekit.linear_regression():fit(features, targets)
  model:save(path)
end

do
  local loaded <close> = sciencekit.load(path)
  assert(loaded.family == "linear-regression")
  assert(loaded.dtype == "f64")

  local prediction = loaded:predict({ { 2.0, 3.0 } })[1][1]
  assert(math.abs(prediction - 14.0) <= 1e-8)
end

os.remove(path)
```

## Supported estimator scope

In `sciencekit_ffi` ABI `1.0` (`sciencekit_lua` `0.5.0`), binary model persistence covers **fitted `f64` linear regression models**:

- `model:save(path)` writes the fitted `linear-regression` parameters to `path` and returns `nil`.
- `sciencekit.load(path)` reads `path` and returns a new `SKLuaModel` userdata with `loaded.family == "linear-regression"` and `loaded.dtype == "f64"`.
- Like any fitted model userdata, a loaded model owns a live C ABI handle (`sciencekit.live_handles()` increments by `1`) and is released via `loaded:close()`, `local loaded <close>`, or garbage collection.

## Persistence error conditions

All persistence failures raise structured Lua runtime errors (see [Structured Errors](./errors.md)):

| Condition | Status name & code | Cause |
| --- | --- | --- |
| Missing file, permission denied, or corrupt payload | `sciencekit io (6)` / `sciencekit invalid-argument (3)` | Propagated from `sk_ffi_model_save` / `sk_ffi_model_load` with the ABI's thread-local error message. |
| Interior NUL byte in `path` (`"model\0.bin"`) | `sciencekit invalid-argument (3)` | Rejected in Rust (`the path contains an interior NUL byte`) before crossing into C. |
| Saving a closed model | `sciencekit invalid-handle (1)` | `the model was already closed`. |
| Saving an unsupported family (`standard-scaler`) | `sciencekit unsupported (4)` | Rejected by the C ABI because persistence currently scopes to `f64` linear regression. |
