# Quickstart

Every example on this page is runnable as written against `lua5.4` once `sciencekit` is on `package.cpath` (see [Installation](./installation.md)). All inputs and assertions come directly from `README.md` and `tests/run.lua`.

## 1. Check module and C ABI metadata

```lua
local sciencekit = require("sciencekit")

print("Module version:", sciencekit.version)          -- "0.5.0"

local abi = sciencekit.abi_version()
print(string.format("ABI version: %d.%d.%d", abi.major, abi.minor, abi.patch))
print("Feature bitmask:", sciencekit.features())
print("Live handles:", sciencekit.live_handles())     -- 0
```

## 2. Linear regression: builder → fit → predict → score

Fit ordinary least squares on the linear relation `y = 1 + 2 * x1 + 3 * x2`:

```lua
local sciencekit = require("sciencekit")

local features = {
  { 0.0, 0.0 },
  { 1.0, 0.0 },
  { 0.0, 1.0 },
  { 2.0, 3.0 },
}
local targets = { 1.0, 3.0, 4.0, 14.0 }

-- Configure the estimator with chainable builder methods, then fit:
local builder = sciencekit.linear_regression()
  :include_intercept(true)
  :backend("faer")

assert(builder.family == "linear-regression")
assert(builder.dtype == "f64")

local model = builder:fit(features, targets)
assert(model.family == "linear-regression")

-- predict() always returns a table of one-element rows: {{y1}, {y2}, ...}
local predictions = model:predict(features)
for index = 1, #predictions do
  print(string.format("row %d -> predicted %.6f (expected %.6f)",
    index, predictions[index][1], targets[index]))
end

-- score() returns the R^2 coefficient of determination as a number
local r_squared = model:score(features, targets)
assert(math.abs(r_squared - 1.0) < 1e-8)
print("R^2:", r_squared)

-- Release native handles deterministically (optional; GC also frees them)
builder:close()
model:close()
```

## 3. Standard scaler: fit and transform

Center each column to zero mean and scale to unit standard deviation:

```lua
local sciencekit = require("sciencekit")

local features = {
  { 0.0, 0.0 },
  { 2.0, 2.0 },
  { 4.0, 4.0 },
  { 6.0, 6.0 },
}

local scaler <close> = sciencekit.standard_scaler()
  :with_mean(true)
  :with_std(true)
  :fit(features)

local transformed = scaler:transform(features)
assert(#transformed == 4 and #transformed[1] == 2)

for row = 1, #transformed do
  print(string.format("row %d -> [%.4f, %.4f]",
    row, transformed[row][1], transformed[row][2]))
end
-- Symmetric rows sum to zero after centering:
assert(math.abs(transformed[1][1] + transformed[4][1]) < 1e-9)
```

<div class="sk-box sk-box--tip">
  <p><strong>Automatic cleanup with <code>&lt;close&gt;</code>:</strong> Marking a builder or model with Lua 5.4's <code>local scaler &lt;close&gt; = ...</code> attribute invokes its <code>__close</code> metamethod automatically when the scope exits — even if an error is raised.</p>
</div>

## Next steps

- Learn all builder options and prediction layouts in [Linear Regression](./guides/linear-regression.md) and [Standard Scaler](./guides/standard-scaler.md).
- Save fitted regression models to disk and reload them in [Model Persistence](./guides/persistence.md).
- Branch on stable error codes with `pcall` in [Structured Errors](./guides/errors.md).
- Understand deterministic cleanup and leak assertions in [Handle Lifecycle](./guides/lifecycle.md).
