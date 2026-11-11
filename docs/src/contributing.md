# Contributing

Contributions to [`neurono-ml/sciencekit-lua`](https://github.com/neurono-ml/sciencekit-lua) follow the same engineering standards as the main [`sciencekit`](https://github.com/neurono-ml/sciencekit) repository. Everything durable in the repository — code, comments, tests, documentation, commits, issues, and pull requests — is written in **English**.

## Repository layout

```text
sciencekit-lua/
├── Cargo.toml                      # Standalone crate (Rust 1.85, edition 2024)
├── sciencekit-0.5.0-1.rockspec     # LuaRocks 3.0 packaging definition
├── src/
│   ├── lib.rs                      # Module table & luaopen_sciencekit entry point
│   ├── error.rs                    # SKLuaError formatting & SKFfiStatus mapping
│   ├── lifecycle.rs                # Thread-local live-handle counter
│   ├── array/                      # Pure dispatcher + Lua table <-> Vec<f64> copy
│   ├── engine/                     # Safe RAII wrappers over sciencekit_ffi handles
│   ├── userdata/                   # SKLuaBuilder and SKLuaModel mlua UserData
│   └── *_tests.rs                  # Companion Rust unit & integration test modules
├── tests/
│   └── run.lua                     # Standalone Lua 5.4 smoke suite
└── docs/                           # mdBook documentation site & SEO tooling
```

## Code conventions

- **No duplicated math:** All numerical work, parameter validation, and model serialization live in `sciencekit` and `sciencekit_ffi`. This crate only converts Lua tables, maps statuses onto `SKLuaError`, and manages userdata lifetimes.
- **Naming:** Public and crate-internal Rust types use the `SK` prefix (`SKLuaBuilder`, `SKLuaMatrix`), and free functions use `sk_` (`sk_lua_matrix_from_table`). Avoid abbreviations in identifiers.
- **Folder modules are pure dispatchers:** `mod.rs` files only declare submodules, re-export items, and wire companion `#[cfg(test)]` modules — never implementation bodies.
- **Companion tests:** Keep tests in sibling `*_tests.rs` files declared at the bottom of the module (`lib_tests.rs`, `error_tests.rs`, `input_tests.rs`, `lifecycle_tests.rs`, `gc_tests.rs`, `operations_tests.rs`, `persistence_tests.rs`, `concurrency_tests.rs`, `large_tests.rs`), plus end-to-end coverage in `tests/run.lua`.

## Local verification gates

Run every CI gate locally before opening a pull request (matching `.github/workflows/ci.yml`):

```sh
# 1. Verify Cargo.lock is in sync
cargo metadata --locked --format-version 1 > /dev/null

# 2. Check formatting
cargo fmt --all --check

# 3. Run Clippy with warnings denied
cargo clippy --all-targets -- -D warnings

# 4. Build release cdylib + rlib and run embedded Lua 5.4 tests
cargo build --release
cargo test

# 5. Run the standalone Lua 5.4 smoke suite (when lua5.4 is installed)
SCIENCEKIT_LUA_LIB=target/release/libsciencekit_lua.so lua5.4 tests/run.lua
```

## Building the documentation locally

Install [`mdBook`](https://rust-lang.github.io/mdBook/) (`v0.5.2`) and Node.js, then build the book and generate `sitemap.xml`:

```sh
mdbook build docs
node docs/scripts/postbuild.mjs
```

`create-missing = false` in `docs/book.toml` ensures any broken chapter path fails the build immediately. Preview locally with live reload using `mdbook serve docs`.

## License

Licensed under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).
