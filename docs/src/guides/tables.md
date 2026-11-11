# Tables and Copy Semantics

Lua has a single composite data structure — the **table** — whose elements are dynamically typed tagged values managed by Lua's garbage collector. Unlike Rust `ndarray` slices or C arrays, a Lua table is never a contiguous unboxed `f64` array in memory.

`sciencekit-lua` states its memory contract plainly: **every input matrix or vector is validated and copied into a contiguous `f64` buffer, and every output is returned as a fresh Lua table of rows.** No zero-copy aliasing is promised across Lua values.

<figure class="sk-diagram reveal">
  <svg viewBox="0 0 820 175" role="img" aria-label="Data flow diagram showing a 1-indexed Lua table of rows validated and copied into a row-major Vec<f64> and SKFfiArray, processed by sciencekit_ffi, and copied into a fresh Lua table of rows before freeing the ABI output buffer">
    <title>Lua table to C ABI buffer copy boundary</title>
    <defs>
      <marker id="arrow-tbl" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
        <path d="M 0 1 L 10 5 L 0 9 z" fill="#64748b" />
      </marker>
    </defs>
    <rect x="16" y="34" width="180" height="96" rx="14" fill="#e0f2fe" stroke="#0284c7" stroke-width="2" />
    <text x="106" y="64" text-anchor="middle" font-size="13" font-weight="700" fill="#0c4a6e">Input Lua Table</text>
    <text x="106" y="86" text-anchor="middle" font-size="11" fill="#0c4a6e">{{1.0, 2.0}, {3, 4.5}}</text>
    <text x="106" y="106" text-anchor="middle" font-size="11" fill="#0369a1">1-indexed · float or integer</text>

    <line x1="196" y1="82" x2="232" y2="82" stroke="#64748b" stroke-width="2" marker-end="url(#arrow-tbl)" />

    <rect x="236" y="34" width="180" height="96" rx="14" fill="#ede9fe" stroke="#7c3aed" stroke-width="2" />
    <text x="326" y="64" text-anchor="middle" font-size="13" font-weight="700" fill="#3b0764">SKLuaMatrix (Copy)</text>
    <text x="326" y="86" text-anchor="middle" font-size="11" fill="#3b0764">row-major Vec&lt;f64&gt;</text>
    <text x="326" y="106" text-anchor="middle" font-size="11" fill="#4c1d95">borrowed SKFfiArray</text>

    <line x1="416" y1="82" x2="452" y2="82" stroke="#64748b" stroke-width="2" marker-end="url(#arrow-tbl)" />

    <rect x="456" y="34" width="145" height="96" rx="14" fill="#f0fdf4" stroke="#16a34a" stroke-width="2" />
    <text x="528" y="64" text-anchor="middle" font-size="13" font-weight="700" fill="#14532d">C ABI Call</text>
    <text x="528" y="86" text-anchor="middle" font-size="11" fill="#14532d">predict / transform</text>
    <text x="528" y="106" text-anchor="middle" font-size="11" fill="#166534">sk_ffi_array_free</text>

    <line x1="601" y1="82" x2="637" y2="82" stroke="#64748b" stroke-width="2" marker-end="url(#arrow-tbl)" />

    <rect x="641" y="34" width="163" height="96" rx="14" fill="#fef3c7" stroke="#d97706" stroke-width="2" />
    <text x="722" y="64" text-anchor="middle" font-size="13" font-weight="700" fill="#78350f">Fresh Output Table</text>
    <text x="722" y="86" text-anchor="middle" font-size="11" fill="#78350f">{{y1}, {y2}, ...}</text>
    <text x="722" y="106" text-anchor="middle" font-size="11" fill="#92400e">no shared memory</text>
  </svg>
  <figcaption>Figure 3 — ABI output buffers are copied into owned Lua tables and freed via <code>sk_ffi_array_free</code> before returning to the script.</figcaption>
</figure>

## Scalar precision (`f64`)

Lua 5.4 numbers come in two subtypes: 64-bit IEEE-754 floats (`Value::Number`) and 64-bit signed integers (`Value::Integer`). During table conversion (`src/array/input.rs`), **both subtypes are accepted** and stored as `f64`:

```lua
-- Mixed integer and float literals convert cleanly to row-major f64:
local features = {
  { 1.0, 2.0 },
  { 3,   4.5 },
  { 5.0, 6.0 },
}
```

Any other Lua type (`string`, `boolean`, `nil`, `table`, `userdata`, `function`) — even a numeric string like `"2.0"` — is rejected with `sciencekit invalid-argument (3)` naming its 1-based coordinates.

## Input matrices (`features`)

Every `features` argument (`fit`, `predict`, `transform`, `score`) must be a **1-indexed rectangular table of row tables**:

- Outer length is read with `raw_len` (`1 .. rows`, ignoring metatables) and must be `>= 1`.
- Row `1` sets the expected column count (`columns >= 1`).
- Every subsequent row `2 .. rows` must be a table with `raw_len == columns`.
- Values are packed in **row-major** order (`row 1 col 1`, `row 1 col 2`, ..., `row N col M`).

## Input target vectors (`targets`)

`targets` passed to `builder:fit(features, targets)` and `model:score(features, targets)` must match `#features` in length (`expected_length = features.rows()`) and may use either of two layouts:

| Layout | Example (`3` samples) | Notes |
| --- | --- | --- |
| Flat sequence | `{ 1.0, 2, 3.5 }` | `table[1]` is a number; elements `1 .. N` must all be numbers. |
| Column of 1-element rows | `{ { 1.0 }, { 2.0 }, { 3.5 } }` | `table[1]` is a table; every row `1 .. N` must be a table of length `1`. Useful when feeding the output of one `predict` directly as targets to another call. |

## Output tables

| Method | Return type | Shape |
| --- | --- | --- |
| `model:predict(features)` | `table` | `N × 1` table of rows: `predictions[row][1]` is a `number`. |
| `model:transform(features)` | `table` | `N × D` table of rows matching `#features` and `#features[1]`. |
| `model:score(features, targets)` | `number` | Scalar `f64` (`R²`). |

Mutating an input table after `:fit(...)` never affects the fitted model, and mutating a returned output table never affects subsequent calls.

## Validation reference (from `src/array/input_tests.rs`)

| Invalid input | Raised error |
| --- | --- |
| `{}` (empty features) | `sciencekit shape (2): features must contain at least one row` |
| `{ {} }` (empty row) | `sciencekit shape (2): feature rows must contain at least one value` |
| `{ { 1.0, 2.0 }, { 3.0 } }` | `sciencekit shape (2): feature row 2 has 1 values, expected 2; a matrix must be rectangular` |
| `{ { 1.0 }, 2.0 }` | `sciencekit invalid-argument (3): row 2 must be a table of numbers, found number` |
| `{ { 1.0, "two" } }` | `sciencekit invalid-argument (3): value at row 1, column 2 must be a number, found string` |
| `targets = { 1.0, 2.0 }` with 3 feature rows | `sciencekit shape (2): targets has 2 values, expected 3 rows` |
| `targets = { { 1.0, 2.0 }, { 3.0, 4.0 } }` | `sciencekit shape (2): target row 1 must contain exactly one value` |
