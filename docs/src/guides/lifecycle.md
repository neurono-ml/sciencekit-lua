# Handle Lifecycle

Every builder (`SKLuaBuilder`) and fitted model (`SKLuaModel`) owns an opaque C ABI pointer (`*mut SKFfiBuilder` or `*mut SKFfiModel`). Because Lua values are garbage-collected while native estimator memory lives outside Lua's heap, `sciencekit-lua` provides **deterministic release** alongside **automatic GC cleanup**, guaranteeing that every handle is freed **exactly once**.

<figure class="sk-diagram reveal">
  <svg viewBox="0 0 820 205" role="img" aria-label="State diagram showing an open handle created by linear_regression, standard_scaler, fit, or load transitioning to Closed exactly once via close(), Lua 5.4 <close> metamethod, or garbage collection Drop">
    <title>Builder and model handle lifecycle</title>
    <defs>
      <marker id="arrow-life" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
        <path d="M 0 1 L 10 5 L 0 9 z" fill="#64748b" />
      </marker>
    </defs>
    <rect x="20" y="50" width="215" height="96" rx="14" fill="#e0f2fe" stroke="#0284c7" stroke-width="2" />
    <text x="127" y="80" text-anchor="middle" font-size="14" font-weight="700" fill="#0c4a6e">Live Handle</text>
    <text x="127" y="102" text-anchor="middle" font-size="12" fill="#0c4a6e">pointer = Some(ptr)</text>
    <text x="127" y="122" text-anchor="middle" font-size="11" fill="#0369a1">handle.closed == false</text>

    <rect x="305" y="18" width="210" height="44" rx="10" fill="#f0fdf4" stroke="#16a34a" stroke-width="1.8" />
    <text x="410" y="45" text-anchor="middle" font-size="12" font-weight="600" fill="#14532d">1. handle:close()</text>

    <rect x="305" y="76" width="210" height="44" rx="10" fill="#ede9fe" stroke="#7c3aed" stroke-width="1.8" />
    <text x="410" y="103" text-anchor="middle" font-size="12" font-weight="600" fill="#3b0764">2. local h &lt;close&gt; (__close)</text>

    <rect x="305" y="134" width="210" height="44" rx="10" fill="#fef3c7" stroke="#d97706" stroke-width="1.8" />
    <text x="410" y="161" text-anchor="middle" font-size="12" font-weight="600" fill="#78350f">3. Lua GC (Rust Drop)</text>

    <line x1="235" y1="76" x2="301" y2="44" stroke="#64748b" stroke-width="1.8" marker-end="url(#arrow-life)" />
    <line x1="235" y1="98" x2="301" y2="98" stroke="#64748b" stroke-width="1.8" marker-end="url(#arrow-life)" />
    <line x1="235" y1="120" x2="301" y2="152" stroke="#64748b" stroke-width="1.8" marker-end="url(#arrow-life)" />

    <line x1="515" y1="44" x2="581" y2="76" stroke="#64748b" stroke-width="1.8" marker-end="url(#arrow-life)" />
    <line x1="515" y1="98" x2="581" y2="98" stroke="#64748b" stroke-width="1.8" marker-end="url(#arrow-life)" />
    <line x1="515" y1="152" x2="581" y2="120" stroke="#64748b" stroke-width="1.8" marker-end="url(#arrow-life)" />

    <rect x="585" y="50" width="215" height="96" rx="14" fill="#ffe4e6" stroke="#e11d48" stroke-width="2" />
    <text x="692" y="80" text-anchor="middle" font-size="14" font-weight="700" fill="#881337">Closed Handle</text>
    <text x="692" y="102" text-anchor="middle" font-size="12" fill="#881337">pointer = None (freed once)</text>
    <text x="692" y="122" text-anchor="middle" font-size="11" fill="#9f1239">further ops → invalid-handle (1)</text>
  </svg>
  <figcaption>Figure 2 — All three release mechanisms funnel through <code>Option::take()</code>, making repeated closes and subsequent GC collection safe no-ops.</figcaption>
</figure>

## Three ways a handle is released

### 1. Explicit `builder:close()` and `model:close()`

Calling `:close()` frees the underlying `SKFfiBuilder` or `SKFfiModel` immediately and sets `.closed = true`. `:close()` is **idempotent**: calling it a second or third time returns `nil` without touching freed memory.

```lua
local sciencekit = require("sciencekit")

local model = sciencekit.linear_regression()
  :fit({ { 1.0 }, { 2.0 }, { 3.0 } }, { 2.0, 4.0, 6.0 })

assert(not model.closed)
assert(tostring(model) == "sciencekit model (linear-regression)")

model:close()
model:close()   -- idempotent no-op
assert(model.closed)
assert(tostring(model) == "sciencekit model (linear-regression) [closed]")
```

### 2. Lua 5.4 to-be-closed variables (`<close>`)

Both `SKLuaBuilder` and `SKLuaModel` implement Lua 5.4's `__close` metamethod (`MetaMethod::Close`). Binding a handle with `local ... <close>` guarantees immediate native memory release when control leaves the lexical block — whether normally, via `break`/`return`, or because an error unwound the stack:

```lua
local sciencekit = require("sciencekit")

do
  local builder <close> = sciencekit.standard_scaler():with_mean(true)
  local model <close> = builder:fit({ { 1.0, 2.0 }, { 3.0, 4.0 } })
  local scaled = model:transform({ { 2.0, 3.0 } })
  assert(#scaled == 1)
end
-- Both `model` and `builder` are already closed here, before GC runs.
```

### 3. Garbage collector (`Drop`)

If neither `:close()` nor `<close>` is used, Lua's garbage collector eventually finalizes unreachable userdata and drops the Rust wrapper (`impl Drop for SKLuaBuilderHandle` / `SKLuaModelHandle`), which invokes `self.close()`. If you already closed the handle explicitly, `Drop` sees `pointer == None` and does nothing.

<div class="sk-box sk-box--info">
  <p><strong>When to close explicitly:</strong> Fitted models hold heap allocations outside Lua's GC accounting, so Lua may not feel memory pressure when a tight loop creates thousands of models. Prefer <code>local model &lt;close&gt; = ...</code> or <code>model:close()</code> inside loops and request handlers.</p>
</div>

## Post-close safety

Attempting any configuration setter, `fit`, `predict`, `transform`, `score`, or `save` on a closed handle fails immediately with `sciencekit invalid-handle (1)`:

```lua
local sciencekit = require("sciencekit")

local builder = sciencekit.linear_regression()
builder:close()

local ok, err = pcall(function()
  builder:include_intercept(true)
end)
assert(not ok)
assert(tostring(err):find("sciencekit invalid-handle (1)", 1, true))
```

## Auditing handle leaks with `sciencekit.live_handles()`

`sciencekit.live_handles()` returns the exact count of `SKLuaBuilderHandle` and `SKLuaModelHandle` instances currently live on the calling OS thread. Combine it with two GC sweeps in test suites to assert zero leaks (from `tests/run.lua`):

```lua
local sciencekit = require("sciencekit")

collectgarbage("collect")
collectgarbage("collect")
local baseline = sciencekit.live_handles()

do
  local model = sciencekit.linear_regression()
    :fit({ { 1.0 }, { 2.0 }, { 3.0 } }, { 2.0, 4.0, 6.0 })
  model:close()
  model = nil
end

collectgarbage("collect")
collectgarbage("collect")
assert(sciencekit.live_handles() == baseline, "every handle was freed once")
```

*(Why does `sciencekit.linear_regression():fit(...)` create two handles briefly? One for the anonymous builder and one for the returned model! Binding `local builder <close>` or running `collectgarbage("collect")` releases the temporary builder too.)*
