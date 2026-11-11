# Concurrency

Lua and Rust have complementary concurrency models, and `sciencekit-lua` respects both boundaries cleanly (tested in `src/concurrency_tests.rs`).

## 1. Lua states are single-threaded (`!Send`)

Standard Lua 5.4 states (`lua_State` / `mlua::Lua`) are not thread-safe and cannot be shared simultaneously across OS threads without external locking. Accordingly:

- `mlua` userdata (`SKLuaBuilder` and `SKLuaModel`) never move between Lua VMs directly.
- `sciencekit.live_handles()` reads a **thread-local** counter (`thread_local! { static LIVE_HANDLES: Cell<usize> }`), because every userdata created on a given OS thread's Lua VM is closed or garbage-collected on that same OS thread.
- Lua coroutines (`coroutine.create` / `coroutine.resume`) run cooperatively on the **same** OS thread and `lua_State`, so using builders and models across Lua coroutines is completely safe.

## 2. Multi-threaded hosts: one Lua VM per worker thread

Host applications written in Rust or C (such as game engines, HTTP servers, or worker pools) achieve multi-core parallelism by spawning one independent Lua 5.4 VM per OS thread and loading `sciencekit` in each VM.

Inside `sciencekit_ffi`, handles are stored in a process-wide mutex-protected registry that hands each call its own `Arc` to the immutable fitted model:

- Fitting, predicting, transforming, and closing handles on Thread A never races with or disturbs handles on Thread B (`concurrency_freeing_one_model_does_not_disturb_another`).
- Multiple independent VMs can fit and score models concurrently in the same process (`concurrency_independent_vms_fit_and_predict`).

### Sharing a trained model across worker VMs

Because Lua userdata values belong to a single `lua_State`, share a trained linear-regression model across worker VMs by saving it once and loading it in each worker:

1. **Training phase:** Fit the model on one VM (or offline) and call `model:save("/path/to/model.state")`.
2. **Worker initialization:** Each worker thread starts its own `lua5.4` VM, calls `local model = require("sciencekit").load("/path/to/model.state")`, and serves `model:predict(features)` requests concurrently without cross-VM contention.

## 3. Native thread safety of fitted models

At the Rust/C ABI layer (`src/engine/model/handle.rs`), `SKLuaModelHandle` implements `Send + Sync`:

- Every read operation (`predict`, `transform`, `score`, `save`) borrows `&self` immutably while `sciencekit_ffi` clones an internal `Arc` from the handle registry for the duration of the call.
- Releasing a handle (`close` / `Drop`) requires `&mut self`, preventing any concurrent read/free race in Rust.
- `concurrency_shared_model_predicts_deterministically` in `src/concurrency_tests.rs` verifies 8 OS threads running 25 concurrent `predict` loops each against a single `Arc<SKLuaModelHandle>` with bit-exact agreement.
