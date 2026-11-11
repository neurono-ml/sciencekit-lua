# Structured Errors

Every failure in `sciencekit-lua` — whether caught during Lua table validation or returned by `sciencekit_ffi` — is raised as a standard Lua runtime error formatted with a deterministic prefix:

```text
sciencekit <status-name> (<code>): <message>
```

Because both `<status-name>` and `<code>` come from `sciencekit_ffi::SKFfiStatus`, scripts can match error categories reliably with `pcall` or `xpcall` without brittle dependencies on human-oriented prose.

## Status code table

From `src/error.rs` and `src/error_tests.rs`, every error maps to one of seven stable status codes:

| Status name | Code | Typical triggers |
| --- | --- | --- |
| `invalid-handle` | `1` | Calling any method or setter on a builder or model after `:close()` or scope exit of `<close>`. |
| `shape` | `2` | Empty tables, ragged matrix rows, empty rows, target length mismatch against `#features`, multi-column target rows, or prediction feature-column count mismatch against a fitted model. |
| `invalid-argument` | `3` | Non-numeric matrix/vector cells, non-table matrix rows, missing `targets` on `linear_regression():fit(...)`, unknown `:backend(...)` name/code, or interior NUL bytes in file paths. |
| `unsupported` | `4` | Calling a method belonging to another estimator family (e.g. `scaler:predict(...)`, `regression:transform(...)`, `regression:with_mean(...)`). |
| `not-fitted` | `5` | Reserved ABI status for operations requiring fitted state. |
| `io` | `6` | Filesystem errors during `model:save(path)` or `sciencekit.load(path)`. |
| `internal` | `7` | Unexpected internal or C ABI invariant failure. |

## Pre-ABI validation vs. C ABI errors

Error reporting works in two layers that produce the exact same prefix format:

1. **Table and argument validation before any ABI call:** `src/array/input.rs` and `src/engine/family.rs` inspect Lua arguments first. Ragged rows and non-numeric cells never cross the FFI boundary, and error messages pinpoint the exact 1-based row and column:
   ```text
   sciencekit shape (2): feature row 2 has 1 values, expected 2; a matrix must be rectangular
   sciencekit invalid-argument (3): value at row 1, column 2 must be a number, found string
   ```
2. **In-process C ABI errors:** When a valid call fails inside `sciencekit_ffi` (for example, `model:predict({{1.0}})` on a model fitted with 2 columns), `SKLuaError::from_last_error` copies the thread-local error message recorded by `sk_ffi_last_error_message()` and prefixes it with `sciencekit <status-name> (<code>):`.

## Catching and parsing errors in Lua

Use `pcall` and pattern-match the header `sciencekit ([%w%-]+) %((%d+)%): (.*)` to extract the status name, numeric code, and detail message:

```lua
local sciencekit = require("sciencekit")

local function parse_sciencekit_error(err)
  local text = tostring(err)
  local status, code, detail = text:match("sciencekit ([%w%-]+) %((%d+)%): (.*)")
  if status then
    return { status = status, code = tonumber(code), message = detail }
  end
  return nil
end

-- 1. Ragged matrix -> shape (2)
local ok1, err1 = pcall(function()
  sciencekit.linear_regression():fit({ { 1.0, 2.0 }, { 3.0 } }, { 1.0, 2.0 })
end)
assert(not ok1)
local info1 = parse_sciencekit_error(err1)
assert(info1.status == "shape" and info1.code == 2)

-- 2. Non-numeric cell -> invalid-argument (3)
local ok2, err2 = pcall(function()
  sciencekit.standard_scaler():fit({ { 1.0, "two" } })
end)
assert(not ok2)
local info2 = parse_sciencekit_error(err2)
assert(info2.status == "invalid-argument" and info2.code == 3)

-- 3. Wrong estimator family -> unsupported (4)
local ok3, err3 = pcall(function()
  local scaler <close> = sciencekit.standard_scaler():fit({ { 1.0 }, { 2.0 } })
  scaler:predict({ { 1.0 } })
end)
assert(not ok3)
local info3 = parse_sciencekit_error(err3)
assert(info3.status == "unsupported" and info3.code == 4)
```

*(Note: Lua's `pcall` may prepend a source location such as `"script.lua:12: "` before `"sciencekit ..."`; avoid anchoring the match with `^` unless you strip the location prefix first.)*
