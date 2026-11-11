# Installation

`sciencekit-lua` is a standalone Rust crate (`edition = "2024"`, `rust-version = "1.85"`) that pulls `sciencekit_ffi` from the main [`sciencekit`](https://github.com/neurono-ml/sciencekit) repository automatically during `cargo build` and compiles Lua 5.4 from vendored C sources.

## Prerequisites

- **Rust 1.85 or newer** (`rustup toolchain install 1.85`) and `cargo`
- **A standard C toolchain** (`gcc` or `clang`) so Cargo's `cc` build script can compile the vendored Lua 5.4 sources
- **Git** (used by Cargo to fetch `sciencekit_ffi`)
- *(Optional)* A **`lua5.4` interpreter** if you want to run standalone `.lua` scripts outside `cargo test`
- *(Optional)* **LuaRocks 3.x** + Lua 5.4 headers (`liblua5.4-dev` or `LUA_INCDIR`) if installing as a system/user rock

<div class="sk-box sk-box--info">
  <p><strong>No system Lua needed to build or test:</strong> Running <code>cargo build --release</code> and <code>cargo test</code> compiles Lua 5.4 directly into the artifact via <code>mlua</code> (<code>lua54</code> + <code>vendored</code>). You do not need <code>liblua</code> or <code>pkg-config</code> installed.</p>
</div>

## Option 1 — Build from source with Cargo

Clone the repository and build the release library:

```sh
git clone https://github.com/neurono-ml/sciencekit-lua.git
cd sciencekit-lua
cargo build --release
```

Run the embedded Rust test suite (which spins up embedded Lua 5.4 VMs in-process):

```sh
cargo test
```

On Linux, `cargo build --release` produces:

```text
target/release/libsciencekit_lua.so
```

(On macOS the artifact is `target/release/libsciencekit_lua.dylib`; on Windows, `target\release\sciencekit_lua.dll`.)

## Option 2 — Install with LuaRocks

The repository includes `sciencekit-0.5.0-1.rockspec` (`rockspec_format = "3.0"`), which builds the crate with `cargo build --release` and installs the compiled library as `$(LIBDIR)/sciencekit.so`:

```sh
luarocks --lua-version=5.4 make sciencekit-0.5.0-1.rockspec
```

Once installed, any `lua5.4` process whose `package.cpath` includes the LuaRocks tree resolves `require("sciencekit")` directly:

```sh
eval "$(luarocks --lua-version=5.4 path)"
lua5.4 tests/run.lua
```

<div class="sk-box sk-box--warning">
  <p><strong>LuaRocks header check:</strong> Even though the module embeds Lua 5.4, the <code>luarocks</code> CLI itself checks for Lua 5.4 C headers (<code>lua.h</code>) while configuring any rock. On Debian/Ubuntu, install <code>liblua5.4-dev</code> (or pass <code>LUA_INCDIR=/path/to/include</code>) when running <code>luarocks make</code>. Direct Cargo builds never need those headers.</p>
</div>

## Loading with `require("sciencekit")` and `package.cpath`

Lua's `require("sciencekit")` searches `package.cpath` for a shared object named **`sciencekit.so`** (not `libsciencekit_lua.so`) and invokes its exported `luaopen_sciencekit` symbol.

### Renaming or symlinking into `package.cpath`

If you built with Cargo and want plain `require("sciencekit")` without LuaRocks, copy or symlink the artifact as `sciencekit.so` in a directory on `package.cpath`:

```sh
mkdir -p ./lua_modules
cp target/release/libsciencekit_lua.so ./lua_modules/sciencekit.so
LUA_CPATH="./lua_modules/?.so;;" lua5.4 -e '
  local sciencekit = require("sciencekit")
  print("sciencekit", sciencekit.version)
'
```

*(Note the trailing `;;` in `LUA_CPATH` — Lua expands `;;` to the default search path instead of replacing it.)*

You can also prepend to `package.cpath` inside Lua before calling `require`:

```lua
package.cpath = "./lua_modules/?.so;" .. package.cpath
local sciencekit = require("sciencekit")
```

### Loading `libsciencekit_lua.so` directly with `package.loadlib`

During local development or CI runs, you can load `target/release/libsciencekit_lua.so` directly without renaming the file by calling `package.loadlib` on `luaopen_sciencekit`. This is the exact helper used by `tests/run.lua`:

```lua
local function load_sciencekit()
  local path = os.getenv("SCIENCEKIT_LUA_LIB")
  if path == nil or path == "" then
    return require("sciencekit")
  end
  local loader, message = package.loadlib(path, "luaopen_sciencekit")
  if loader == nil then
    error("cannot load the native module at " .. path .. ": " .. tostring(message))
  end
  return loader()
end

local sciencekit = load_sciencekit()
```

Run the smoke test suite against a fresh release build:

```sh
SCIENCEKIT_LUA_LIB=target/release/libsciencekit_lua.so lua5.4 tests/run.lua
```

## Pinning the `sciencekit_ffi` revision

By default, `Cargo.toml` tracks the `main` branch of `neurono-ml/sciencekit`:

```toml
sciencekit_ffi = { git = "https://github.com/neurono-ml/sciencekit", branch = "main" }
```

To pin a specific upstream commit or release tag in downstream builds, replace `branch = "main"` with `rev` (or `tag`) and refresh `Cargo.lock`:

```toml
sciencekit_ffi = { git = "https://github.com/neurono-ml/sciencekit", rev = "<commit-sha>" }
```
