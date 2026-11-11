# Introduction

<div class="sk-hero reveal">
  <div class="sk-announce">
    <span class="sk-announce__tag">v0.5.0</span>
    Native Lua 5.4 module over the stable <code>sciencekit_ffi</code> C ABI
  </div>
  <h1 class="sk-hero__title">Machine learning for Lua, backed by Rust</h1>
  <p class="sk-hero__subtitle">
    <strong>sciencekit-lua</strong> compiles the <code>sciencekit</code> native Lua 5.4 module with an embedded Lua interpreter — chainable estimator builders, fitted models, model persistence, structured errors and deterministic handle lifecycle.
  </p>
  <div class="sk-btn-row">
    <a class="sk-btn sk-btn--primary" href="./quickstart.html">Run the Quickstart →</a>
    <a class="sk-btn sk-btn--secondary" href="./api-reference.html">API Reference</a>
    <a class="sk-btn sk-btn--star" href="https://github.com/neurono-ml/sciencekit-lua">GitHub Repository</a>
  </div>
</div>

<div class="sk-stats reveal">
  <div class="sk-stats__item">
    <div class="sk-stats__value">Lua 5.4</div>
    <span class="sk-stats__label">vendored from C source</span>
  </div>
  <div class="sk-stats__item">
    <div class="sk-stats__value">ABI 1.0</div>
    <span class="sk-stats__label">in-process <code>sciencekit_ffi</code></span>
  </div>
  <div class="sk-stats__item">
    <div class="sk-stats__value">f64</div>
    <span class="sk-stats__label">IEEE-754 double precision</span>
  </div>
  <div class="sk-stats__item">
    <div class="sk-stats__value">Apache-2.0</div>
    <span class="sk-stats__label">permissive open-source license</span>
  </div>
</div>

## What is `sciencekit-lua`?

[`sciencekit`](https://github.com/neurono-ml/sciencekit) is a Rust machine-learning library reimplementing scikit-learn's estimators and transformers with native numerical kernels. [`neurono-ml/sciencekit-lua`](https://github.com/neurono-ml/sciencekit-lua) is the standalone Rust crate (`sciencekit_lua`) that packages `sciencekit` as a loadable Lua 5.4 module named `sciencekit`, consumed in scripts with `require("sciencekit")`.

```lua
local sciencekit = require("sciencekit")

local features = { { 0.0, 0.0 }, { 1.0, 0.0 }, { 0.0, 1.0 }, { 2.0, 3.0 } }
local targets = { 1.0, 3.0, 4.0, 14.0 }

local model = sciencekit.linear_regression()
  :include_intercept(true)
  :backend("faer")
  :fit(features, targets)

local predictions = model:predict(features)   -- table of one-element rows
local r_squared = model:score(features, targets)
```

## Embedded Lua 5.4 — no system Lua required

Native scripting bindings often fail at build time because the host machine lacks `liblua`, `lua.h`, or a matching `pkg-config` definition. `sciencekit_lua` eliminates that dependency by configuring [`mlua`](https://crates.io/crates/mlua) `0.10` with `features = ["lua54", "vendored"]`:

- **Compiled from source:** Cargo compiles the official Lua 5.4 C sources directly into the Rust build using a standard C compiler (`cc`). No system `liblua`, header package, or `pkg-config` file is needed to run `cargo build --release` or `cargo test`.
- **Hand-written module entry point:** Because `mlua`'s `#[mlua::lua_module]` attribute macro (`module` feature) refuses to coexist with `vendored`, `src/lib.rs` exports `luaopen_sciencekit` explicitly as an `extern "C-unwind"` function backed by `mlua::Lua::entrypoint1`. When a host `lua5.4` interpreter loads the shared library, `luaopen_sciencekit` populates the module table against the caller's `lua_State`.
- **Self-contained Rust tests:** The crate produces both `cdylib` (for `require("sciencekit")` in external Lua 5.4 processes) and `rlib` (so `cargo test` spins up fresh embedded Lua 5.4 VMs in memory without shelling out to a `lua` binary).

## How the module wraps the Rust core

Rather than re-implementing estimator configuration, parameter validation, fitting, or binary serialization in a binding-specific layer, `sciencekit_lua` links `sciencekit_ffi` from the main `sciencekit` repository in-process via a Cargo git dependency. Every Lua call traverses the exact same C ABI handles, descriptors, and status codes that a C or Go consumer sees.

<figure class="sk-diagram reveal">
  <svg viewBox="0 0 860 210" role="img" aria-label="Architecture diagram showing a Lua 5.4 script calling require('sciencekit'), passing through mlua userdata and table conversion into in-process sciencekit_ffi C ABI entry points and Rust estimators">
    <title>sciencekit-lua architecture over the in-process C ABI</title>
    <defs>
      <marker id="arrow-arch" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
        <path d="M 0 1 L 10 5 L 0 9 z" fill="#64748b" />
      </marker>
    </defs>
    <rect x="16" y="34" width="175" height="92" rx="14" fill="#e0f2fe" stroke="#0284c7" stroke-width="2" />
    <text x="103" y="68" text-anchor="middle" font-size="14" font-weight="700" fill="#0c4a6e">Lua 5.4 Script</text>
    <text x="103" y="90" text-anchor="middle" font-size="12" fill="#0c4a6e">require("sciencekit")</text>
    <text x="103" y="109" text-anchor="middle" font-size="11" fill="#0369a1">tables + &lt;close&gt; locals</text>

    <line x1="191" y1="80" x2="231" y2="80" stroke="#64748b" stroke-width="2" marker-end="url(#arrow-arch)" />

    <rect x="235" y="22" width="205" height="116" rx="14" fill="#ede9fe" stroke="#7c3aed" stroke-width="2" />
    <text x="337" y="54" text-anchor="middle" font-size="14" font-weight="700" fill="#3b0764">sciencekit_lua</text>
    <text x="337" y="76" text-anchor="middle" font-size="12" fill="#3b0764">SKLuaBuilder / SKLuaModel</text>
    <text x="337" y="96" text-anchor="middle" font-size="11" fill="#4c1d95">Table ↔ Vec&lt;f64&gt; copy</text>
    <text x="337" y="116" text-anchor="middle" font-size="11" fill="#4c1d95">SKLuaError + Drop / close</text>

    <line x1="440" y1="80" x2="480" y2="80" stroke="#64748b" stroke-width="2" marker-end="url(#arrow-arch)" />

    <rect x="484" y="34" width="175" height="92" rx="14" fill="#f0fdf4" stroke="#16a34a" stroke-width="2" />
    <text x="571" y="68" text-anchor="middle" font-size="14" font-weight="700" fill="#14532d">sciencekit_ffi</text>
    <text x="571" y="90" text-anchor="middle" font-size="12" fill="#14532d">Stable C ABI (in-process)</text>
    <text x="571" y="109" text-anchor="middle" font-size="11" fill="#166534">SKFfiArray · SKFfiStatus</text>

    <line x1="659" y1="80" x2="699" y2="80" stroke="#64748b" stroke-width="2" marker-end="url(#arrow-arch)" />

    <rect x="703" y="34" width="141" height="92" rx="14" fill="#fef3c7" stroke="#d97706" stroke-width="2" />
    <text x="773" y="68" text-anchor="middle" font-size="14" font-weight="700" fill="#78350f">Rust Core</text>
    <text x="773" y="90" text-anchor="middle" font-size="12" fill="#78350f">faer · ndarray</text>
    <text x="773" y="109" text-anchor="middle" font-size="11" fill="#92400e">Fitted estimators</text>

    <rect x="235" y="154" width="424" height="40" rx="10" fill="#ffe4e6" stroke="#e11d48" stroke-width="1.5" />
    <text x="447" y="179" text-anchor="middle" font-size="12" font-weight="600" fill="#881337">Failures return "sciencekit &lt;status-name&gt; (&lt;code&gt;): &lt;message&gt;" to pcall</text>
  </svg>
  <figcaption>Figure 1 — Every Lua call converts tables into contiguous <code>f64</code> buffers and invokes <code>sciencekit_ffi</code> in-process.</figcaption>
</figure>

The binding layer has four focused responsibilities:

<div class="sk-cards reveal">
  <div class="sk-card">
    <span class="sk-card__icon sk-icon--indigo">1</span>
    <div class="sk-card__title">Idiomatic Userdata</div>
    <p>Wraps opaque C handles (<code>SKFfiBuilder</code> and <code>SKFfiModel</code>) in chainable Lua userdata with inspectable <code>family</code>, <code>dtype</code>, and <code>closed</code> fields.</p>
  </div>
  <div class="sk-card">
    <span class="sk-card__icon sk-icon--sky">2</span>
    <div class="sk-card__title">Validated Table Copies</div>
    <p>Checks rectangularity and numeric cells up front, copies Lua tables into contiguous row-major <code>f64</code> buffers, and returns fresh Lua tables on output.</p>
  </div>
  <div class="sk-card">
    <span class="sk-card__icon sk-icon--rose">3</span>
    <div class="sk-card__title">Structured Errors</div>
    <p>Maps both pre-call table errors and C ABI thread-local messages into Lua runtime errors prefixed by <code>sciencekit &lt;status&gt; (&lt;code&gt;):</code>.</p>
  </div>
  <div class="sk-card">
    <span class="sk-card__icon sk-icon--emerald">4</span>
    <div class="sk-card__title">Single-Free Lifecycle</div>
    <p>Frees native handles deterministically via <code>:close()</code> or Lua 5.4 <code>&lt;close&gt;</code> variables, with garbage-collector fallback and zero double-frees.</p>
  </div>
</div>

## Scope and explicit non-goals

| Dimension | Current behavior in `0.5.0` |
| --- | --- |
| **Estimator families** | Ordinary least-squares linear regression (`sciencekit.linear_regression()`) and column standardization (`sciencekit.standard_scaler()`), matching the C ABI surface. |
| **Persistence** | `model:save(path)` and `sciencekit.load(path)` for fitted `f64` linear regression models. |
| **Scalar type** | `f64` only (`dtype == "f64"`). Lua 5.4 numbers are IEEE-754 doubles, so `f32` descriptors would add rounding without saving Lua memory. Both Lua floats and integers are accepted as input cells. |
| **Memory model** | Explicit copy semantics in both directions. Lua tables are garbage-collected hash/array hybrids, so no zero-copy aliasing is promised across the Lua boundary. |
| **Non-goals** | LuaJIT, Lua 5.1–5.3, async Lua coroutines over Tokio, and GPU-specific Lua APIs. |
