# API Reference

Complete reference for the `sciencekit` Lua 5.4 module (`sciencekit_lua` `0.5.0`), grounded directly in `src/lib.rs`, `src/userdata/builder.rs`, `src/userdata/model.rs`, `src/engine/family.rs`, `src/error.rs`, and `tests/run.lua`.

```lua
local sciencekit = require("sciencekit")
```

`require("sciencekit")` always returns a plain Lua `table` (`type(sciencekit) == "table"`), never a callable proxy.

---

## Module table (`sciencekit`)

| Export | Kind | Signature | Description |
| --- | --- | --- | --- |
| `sciencekit.version` | `string` field | `string` | Package version compiled into the crate (`"0.5.0"`). Note: this is a string value, not a function. |
| `sciencekit.abi_version` | function | `() -> { major: integer, minor: integer, patch: integer }` | Returns a fresh table carrying `SK_FFI_ABI_MAJOR`, `SK_FFI_ABI_MINOR`, and `SK_FFI_ABI_PATCH` (currently `{ major = 1, minor = 0, patch = 0 }`). |
| `sciencekit.features` | function | `() -> integer` | Returns the `sk_ffi_features()` capability bitmask reported by `sciencekit_ffi` as a 64-bit Lua integer. |
| `sciencekit.live_handles` | function | `() -> integer` | Returns the number of live (not yet closed or garbage-collected) builder and model handles on the calling OS thread. |
| `sciencekit.linear_regression` | function | `() -> Builder` | Allocates an `f64` ordinary least-squares builder userdata (`family == "linear-regression"`). |
| `sciencekit.standard_scaler` | function | `() -> Builder` | Allocates an `f64` column-standardization builder userdata (`family == "standard-scaler"`). |
| `sciencekit.load` | function | `(path: string) -> Model` | Loads a persisted fitted `f64` linear-regression model from `path` (`loaded.family == "linear-regression"`). |

### Native symbol

| C symbol | Signature | Description |
| --- | --- | --- |
| `luaopen_sciencekit` | `unsafe extern "C-unwind" fn(*mut lua_State) -> c_int` | Native Lua 5.4 module entry point used by `require("sciencekit")` and `package.loadlib(path, "luaopen_sciencekit")`. |

---

## Builder userdata (`SKLuaBuilder`)

Returned by `sciencekit.linear_regression()` and `sciencekit.standard_scaler()`.

### Read-only fields

| Field | Type | Values | Description |
| --- | --- | --- | --- |
| `builder.family` | `string` | `"linear-regression"` \| `"standard-scaler"` | Estimator family identifier. |
| `builder.dtype` | `string` | `"f64"` | Scalar precision (`f64` IEEE-754 double). |
| `builder.closed` | `boolean` | `false` \| `true` | `true` once `:close()` or `__close` has released the handle. |

### Methods

All configuration setters mutate the builder in place and return the **same** builder userdata (`self`), enabling fluent chaining.

#### `builder:include_intercept(enabled) -> builder`
- **Parameters:** `enabled: boolean`
- **Supported families:** `"linear-regression"` (raises `sciencekit unsupported (4)` on `"standard-scaler"`).
- **Description:** Configures whether linear regression fits an intercept term.

#### `builder:with_mean(enabled) -> builder`
- **Parameters:** `enabled: boolean`
- **Supported families:** `"standard-scaler"` (raises `sciencekit unsupported (4)` on `"linear-regression"`).
- **Description:** Configures whether each column is centered by subtracting its training mean.

#### `builder:with_std(enabled) -> builder`
- **Parameters:** `enabled: boolean`
- **Supported families:** `"standard-scaler"` (raises `sciencekit unsupported (4)` on `"linear-regression"`).
- **Description:** Configures whether each column is divided by its training standard deviation.

#### `builder:backend(backend) -> builder`
- **Parameters:** `backend: string | integer`
- **Supported families:** `"linear-regression"`, `"standard-scaler"`
- **Accepted values:**
  - `"faer"` or `0`
  - `"matrix-multiply"` (alias `"matrixmultiply"`) or `1`
  - `"ndarray-linalg"` (alias `"ndarray_linalg"`) or `2`
- **Errors:** Raises `sciencekit invalid-argument (3)` if given an unknown string, an integer outside `0..2`, or any other Lua type.

#### `builder:fit(features[, targets]) -> Model`
- **Parameters:**
  - `features: table` — non-empty rectangular 1-indexed table of numeric rows (`N × D`).
  - `targets: table | nil` — required when `builder.family == "linear-regression"` (flat sequence `{ y1, ..., yN }` or column `{ {y1}, ..., {yN} }` with length `N`); omitted (`nil`) for `"standard-scaler"`.
- **Returns:** A new fitted `Model` userdata (`model.family == builder.family`, `model.dtype == "f64"`). The builder itself remains open.
- **Errors:** Raises `sciencekit shape (2)`, `sciencekit invalid-argument (3)`, or `sciencekit invalid-handle (1)`.

#### `builder:close() -> nil`
- **Description:** Releases the underlying `SKFfiBuilder` handle immediately and marks `builder.closed = true`. Idempotent: subsequent calls are no-ops returning `nil`.

### Metamethods

| Metamethod | Output / Behavior |
| --- | --- |
| `__close` | Releases the handle when a `local builder <close> = ...` variable leaves scope. Idempotent. |
| `__tostring` | `"sciencekit builder (<family>)"` when open; `"sciencekit builder (<family>) [closed]"` when closed. |

---

## Fitted model userdata (`SKLuaModel`)

Returned by `builder:fit(...)` and `sciencekit.load(path)`.

### Read-only fields

| Field | Type | Values | Description |
| --- | --- | --- | --- |
| `model.family` | `string` | `"linear-regression"` \| `"standard-scaler"` | Estimator family identifier. |
| `model.dtype` | `string` | `"f64"` | Scalar precision (`"f64"`). |
| `model.closed` | `boolean` | `false` \| `true` | `true` once `:close()` or `__close` has released the handle. |

### Methods

#### `model:predict(features) -> table`
- **Parameters:** `features: table` — rectangular `N × D` table of numbers matching the fitted column count `D`.
- **Supported families:** `"linear-regression"` (raises `sciencekit unsupported (4)` on `"standard-scaler"`).
- **Returns:** Fresh `N × 1` Lua table of one-element rows: `{ { y1 }, { y2 }, ..., { yN } }`.

#### `model:transform(features) -> table`
- **Parameters:** `features: table` — rectangular `N × D` table of numbers matching the fitted column count `D`.
- **Supported families:** `"standard-scaler"` (raises `sciencekit unsupported (4)` on `"linear-regression"`).
- **Returns:** Fresh `N × D` Lua table of transformed rows.

#### `model:score(features, targets) -> number`
- **Parameters:**
  - `features: table` — rectangular `N × D` table of numbers.
  - `targets: table` — flat sequence or `N × 1` table of rows with length `N`.
- **Supported families:** `"linear-regression"` (raises `sciencekit unsupported (4)` on `"standard-scaler"`).
- **Returns:** Coefficient of determination (`R²`) as a Lua `number` (`f64`).

#### `model:save(path) -> nil`
- **Parameters:** `path: string` — filesystem path without interior NUL bytes.
- **Supported families:** `"linear-regression"` (raises `sciencekit unsupported (4)` on `"standard-scaler"`).
- **Returns:** `nil`.

#### `model:close() -> nil`
- **Description:** Releases the underlying `SKFfiModel` handle immediately and marks `model.closed = true`. Idempotent.

### Metamethods

| Metamethod | Output / Behavior |
| --- | --- |
| `__close` | Releases the handle when a `local model <close> = ...` variable leaves scope. Idempotent. |
| `__tostring` | `"sciencekit model (<family>)"` when open; `"sciencekit model (<family>) [closed]"` when closed. |

---

## Error format & status codes

Runtime error strings always match:

```text
sciencekit <status-name> (<code>): <message>
```

| Status name | Code |
| --- | --- |
| `invalid-handle` | `1` |
| `shape` | `2` |
| `invalid-argument` | `3` |
| `unsupported` | `4` |
| `not-fitted` | `5` |
| `io` | `6` |
| `internal` | `7` |
