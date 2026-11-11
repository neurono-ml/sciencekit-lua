# Frequently Asked Questions

## Why does `sciencekit-lua` embed Lua 5.4 instead of linking the system `liblua`?

Linking dynamically against a system `liblua` forces every build machine and CI image to carry matching headers (`lua.h`), `pkg-config` metadata, and compatible symbol exports. Enabling `mlua` with `features = ["lua54", "vendored"]` compiles Lua 5.4 directly from source with Cargo's `cc` driver so `cargo build --release` and `cargo test` work out of the box on a clean Rust toolchain.

## If Lua is vendored, why does `luarocks make` still ask for Lua 5.4 headers?

That check belongs to the `luarocks` package manager itself, not to `cargo build`. Before invoking any rockspec's `build_command`, LuaRocks validates its own local configuration by probing for `lua.h` in `LUA_INCDIR` (provided by packages like `liblua5.4-dev`). If you build with `cargo build --release` and copy `libsciencekit_lua.so` to `sciencekit.so` on `package.cpath`, no Lua headers are needed anywhere.

## Does this module work with LuaJIT or Lua 5.1 / 5.2 / 5.3?

No. Lua 5.1–5.3 and LuaJIT have distinct C ABIs and lack Lua 5.4 features such as to-be-closed variables (`local model <close> = ...`). Those runtimes are explicit non-goals for `sciencekit-lua`.

## Why does `model:predict(features)` return `{{y1}, {y2}, ...}` instead of `{y1, y2, ...}`?

Both `predict` and `transform` convert C ABI output descriptors into a uniform 2-dimensional Lua representation — a table of rows (`src/engine/model/output.rs` and `src/array/output.rs`). For single-target regression, each sample's prediction sits at `predictions[row][1]`. Notably, that `N × 1` table of one-element rows is also directly accepted as a `targets` argument by `fit` and `score`.

## Why are tables copied instead of passed zero-copy?

In standard Lua 5.4, a table is a garbage-collected structure of tagged `TValue` slots rather than a dense IEEE-754 buffer. Passing data into BLAS/SIMD Rust kernels (`faer`, `ndarray`) requires contiguous row-major `f64` slices, so copying and validating on entry — and copying outputs into fresh Lua tables before freeing ABI buffers — gives predictable performance and memory safety without pinning Lua GC objects.

## Why is `dtype` always `"f64"`?

Every floating-point number in standard Lua 5.4 is already a 64-bit IEEE-754 `double`. Converting Lua numbers down to `f32` and back to `f64` on return would introduce precision loss without reducing the memory footprint of any Lua table.

## Which estimators are available, and how are new ones added?

`sciencekit-lua` exposes every estimator family carried by the shared `sciencekit_ffi` C ABI — currently `linear_regression` and `standard_scaler`. New algorithms land first in the Rust core (`neurono-ml/sciencekit`), gain C ABI entry points in `sciencekit_ffi`, and are then exposed here as thin userdata wrappers without duplicating numerical logic.
