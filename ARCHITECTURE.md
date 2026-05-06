# Architecture

How `index.html` is organized inside. Read this before editing the script.

The codebase is **procedural** — module-pattern singletons + render functions + a single refresh loop. No class hierarchy. There's a thin `window.CairoAtlas` devtools facade exposed at the bottom of the script, but it's pure delegation; the architectural spine is the singletons and `refresh()`.

---

## One-screen mental model

```
                ┌────────────────────────────────────────────┐
                │              window.cowork                 │  artifact runtime
                │   callMcpTool(name, args)                  │
                │   askClaude(prompt, attachments)           │
                └──────────┬───────────────────┬─────────────┘
                           │                   │
                  (MCP data calls)      (chat completions)
                           │                   │
                ┌──────────▼──────────┐  ┌─────▼──────────────┐
                │   ATLAS singleton   │  │   CHAT singleton   │
                │  rate-limit · log   │  │  history · send    │
                └──────────┬──────────┘  └────────┬───────────┘
                           │                      │
                ┌──────────▼─────────────────────────────────┐
                │              fetchAll()                    │
                │   17 parallel callTool() invocations       │
                └──────────┬─────────────────────────────────┘
                           │
                ┌──────────▼─────────────────────────────────┐
                │            buildCache(data)                │
                │  pure aggregator → STATE.cache.{...}       │
                └──────────┬─────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
       ┌──────▼──────┐           ┌──────▼──────┐
       │ ChartManager│           │  Renderers  │
       │  (lifecycle │           │  (KPIs,     │
       │  + reconcile│           │   tables,   │
       │  for slot   │           │   funnel,   │
       │   swaps)    │           │   content)  │
       └─────────────┘           └─────────────┘
                                        │
                                  ┌─────▼──────┐
                                  │    DOM     │
                                  └────────────┘
```

The whole thing runs on a `setTimeout` loop driven by `STATE.intervalSec`.

---

## Module map (singletons + procedural functions)

The script splits into 12 logical sections (listed at the top of the script body as a ToC comment).

### Boot diagnostics — `__atlasBanner`, `probeCoworkRuntime` (top of script)
Fixed-position banners injected into `<body>` for things the user must see even if no other paint succeeds: missing Cowork runtime, Chart.js CDN failure. Idempotent. Called once from the init IIFE.

### Error surface — `atlasLog`, `safeRender`, `$`, `on` (top of script)
- `window.onerror` + `unhandledrejection` → `atlasLog('error', ...)` → `#error-log` panel.
- `safeRender(name, fn)` wraps each renderer in `refresh()` so one render failure logs and the rest of the pass continues.
- `$(id)` / `on(id, ev, h)` defensive DOM lookups — missing element warns instead of throwing.
- `setHtmlIfChanged(id, html)` idempotent innerHTML write to suppress repaint flash.

### `ATLAS_CONFIG` (frozen object near top of script body)
The one place to edit when forking. Holds:
- `mcp.{shopifyAnalytics, igReels, ttVideos, caInsights, caSummary, caTranscript}` — MCP tool names.
- `metricoolBlogId`, `shopTz`, `storagePrefix` — brand/locale knobs.

Existing `TOOL_SHOPIFY` / `TOOL_IG_REELS` / `TOOL_TT_VIDEOS` / `METRICOOL_BLOG_ID` / `SHOP_TZ` are now thin aliases off this config. Exposed as `window.ATLAS_CONFIG`.

### `ATLAS` singleton
Rate-limited MCP wrapper + log buffer.
- `ATLAS.api` — `{inflight, maxConcurrent: 5, minIntervalMs: 80, lastCallAt, counters: {total, errors, slow}}`.
- `ATLAS.logs` — capped at 60 entries, FIFO.
- `atlasGate()` yields until inflight slot + global throttle clear.
- `atlasApiCall(toolName, args, opts)` — full pipeline: gate → race against 16s timeout → log slow/error → release.
- `callTool(name, args)` — the simple wrapper used by `fetchAll()`. Short-circuits with a friendly error when `window.cowork` is missing instead of throwing on undefined. Catches and returns `{__error: msg}` so promises never reject.

> **Known sharp edge.** `fetchAll()` uses `callTool()` directly, not `atlasApiCall()` — so the `maxConcurrent: 5` ceiling is *not* enforced for the actual hot path. Acceptable for current MCP host; would bite under stricter throttling.

### `STATE` singleton
All dashboard state:
```js
STATE = {
  range, custom,                    // active date range (preset id + optional custom dates)
  intervalSec, paused, timer,       // refresh loop control
  lastRefreshedAt,                  // Date — drives the freshness pill
  sort: {channel, source, medium, campaigns},   // {col, dir} per table
  cache: { ... slices },            // see "Cache shape" below
  insightHistory                    // CA history (currently unused outside CA)
}
```
Note: chart instances live in `ChartManager._charts`, not in `STATE`.

### Channel classifiers (pure functions)
- `bucketSessionChannel(src, med, refSrc, refName)` — sessions side: uses `referrer_source` + `referrer_name`.
- `bucketOrderChannel(src, med, refChan)` — orders side: uses Shopify's `referring_channel`.
- `variantSuffix(src)` — turns `igstory`/`igreel`/`igbio`/`tt*`/`fb*` into the variant tail.

Order of precedence: Meta paid → IG/TT/FB organic with variant → Klaviyo email → Google organic → social referrer fallback → Direct → Other.

### `fetchAll()` + `buildCache()` — data layer
- `fetchAll()` — fires 10 Shopify queries + 2 Metricool calls + 5 prev-period Shopify + 2 prev-period Metricool in parallel via `Promise.all`. Returns `{key: rawResult, ...}`. Each call wrapped in `callTool()` which catches and returns `{__error: msg}`, so the promise never rejects.
- `buildCache(data)` — pure transform. Produces these slices on `STATE.cache`:

```
STATE.cache = {
  channel,        // [{label, sess, orders, rev, cvr}], with channelTotal
  source,         // top-12 [{label, sess, orders, rev, cvr}]
  medium,         // top-12 by medium
  campaigns,      // top-15 [{camp, src, med, sess, orders, rev, cvr}]
  matrix,         // {srcs, meds, cells: {"src|med": n}, max}
  coverage,       // {total, taggedSum, segs:[{label, val, color}]}
  kpi,            // {sessions, taggedSess, orders, revenue, cvr (fraction)}
  funnel,         // {sessions, cart, checkout, completed}
  volume,         // {gran, rows: [{ts, sessions, orders, revenue}]}
  posts,          // [{net:'IG'|'TT', timeIso, content, stat1{lbl,val}, stat2, stat3, ...}]
  postsRange,     // {from, to, label, fallback}
  contentFunnel,  // {total, ig, tt, fb, metricoolRange}
  contentDaily,   // {range:{gran,label}, sessRows, ordersRows, reachByDay}
  prev            // previous-period mirror for delta arrows
}
```

### `ChartManager` (module-pattern object)
Centralized Chart.js lifecycle with self-healing cache.
- `ChartManager.render(canvasId, configFn, updateFn)` — verifies the cached instance is still bound to the live canvas; mismatch → destroy + recreate. Eliminated a class of bug where empty-state paths leaked stale chart references against detached canvases.
- `ChartManager.destroy(canvasId)` — empty-state paths call this instead of manually nulling caches.
- `ChartManager.reconcile()` — runs after `LayoutManager` swaps to flush charts whose canvases have left the DOM.

All four chart renderers (`renderReachDonut`, `renderPostBars`, `renderContentChart`, `renderVolumeChart`) go through this.

### `LayoutManager` (module-pattern object)
Slot-discovery layout engine.
- Every draggable element declares `data-slot="<id>"` + `data-slot-group="<group>"`.
- `LayoutManager._scanGroups()` finds all `[data-slot][data-slot-group]` elements and groups them by `groupId @ parentSelector`. Slots can only swap with peers in the same group AND with the same parent.
- On boot: scan DOM, apply saved order from `localStorage('cairo_atlas_layout_v1')`, wire drag handlers per slot.
- Adding a new draggable peer = put `<div ... data-slot="..." data-slot-group="...">` in HTML. No JS change needed.
- Reset via the topbar "Reset layout" button (clears the storage key and reloads).
- Drag handle is a `<button draggable="true">`. The slot itself is the drop target. Touch devices: handles auto-hide via `@media (hover: none)`.

### Renderers (functions, ~line 2200–3000)
One per UI section:
- `renderKpis`, `renderChannelTable`, `renderSourceTable`, `renderMediumTable`, `renderCampaignsTable`, `renderMatrix`, `renderCoverage`, `renderFunnel`, `renderContentFunnel`, `renderVolumeChart`, `renderReachDonut`, `renderPostBars`, `renderContentChart`, `renderPosts`.

Conventions:
- Each reads from `STATE.cache.<slice>` and writes into a fixed DOM id.
- Sortable tables share `thHTML(col, label, sortKey, sort, align)` for header rendering and `sortRows(rows, col, dir)` for ordering.
- Bars are CSS divs (`.bar-track > .bar-fill`) sized by `(value / max) * 100%`.
- Channel tables append a `totalrow`; campaigns table does not (intentional).
- Every renderer is invoked through `safeRender('renderName', fn)` in `refresh()`.

### Creative Analysis — `CA` singleton (~line 2800)
Heavier subsystem (7+ API calls per analysis), manually triggered by the `#ca-load-btn` button. Owns its own state (`CA.currentSlide`, slide content, transcripts), its own renderers (`caRender`, `caRenderSlide`, `caRenderReasoning`), its own progress UI. Picks Best/Underrated/Worst from Motion's last-30d Meta spend table.

### Date range picker — `DR` singleton (~line 3700)
Custom calendar (Mon-first, range select, future-day disable, prev/next month nav, preset buttons). Apply writes `STATE.range = 'custom'` + `STATE.custom = {from, to}`, persists to localStorage, and triggers `refresh()`.

### Chat — `CHAT` singleton (~line 3950)
Floating glass-FAB Cairo AI chat. Compresses `STATE.cache` into a slim payload (`compressCacheForChat()`), calls `window.cowork.askClaude(prompt, [data])`, renders the conversation, generates 3 dynamic suggestion prompts in the background (`ensureSuggestions()`). Persists history (last 50), text size, window size, and suggestions to localStorage.

### `window.CairoAtlas` (devtools facade, end of script)
Thin class wrappers (`AtlasApi`, `Dashboard`, `ChatBubble`, `DateRangePicker`) that delegate to the procedural functions and singletons above. Pure navigability aid — open devtools, type `CairoAtlas.dashboard.refresh()` instead of digging for the function name. **Not the architectural spine; not used internally.** If you cut it, the dashboard still works.

---

## Refresh loop in detail

```
setRange / setIntervalSec / pause toggle
            │
            ▼
       refresh()                  guarded by _refreshing flag
            │
            ├─ setFreshness('loading')
            ├─ startFabLoading(2800)
            ├─ setRefreshLoading(true)
            ├─ fetchAll()                   17 MCP calls in parallel
            ├─ checkErrors(data)            >=12 errors → bail with 'error' freshness
            ├─ updateDateLabel()
            ├─ safeRender('buildCache', …)
            ├─ safeRender('renderKpis', …)
            ├─ safeRender('renderVolumeChart', …)   via ChartManager
            ├─ safeRender('renderChannelTable', …)
            ├─ safeRender('renderSourceTable', …)
            ├─ safeRender('renderMediumTable', …)
            ├─ safeRender('renderCampaignsTable', …)
            ├─ safeRender('renderMatrix', …)
            ├─ safeRender('renderCoverage', …)
            ├─ safeRender('renderFunnel', …)
            ├─ safeRender('renderContentFunnel', …)
            ├─ safeRender('renderPosts', …)
            ├─ STATE.lastRefreshedAt = new Date()
            └─ setFreshness('fresh')
            │
            ▼
       scheduleNext()                       setTimeout(STATE.intervalSec * 1000)
```

---

## Error handling philosophy

> "Fail loud, log to panel, never freeze."

1. **Boot-level** banner (`__atlasBanner`) for things that block everything else: missing Cowork runtime, missing Chart.js global. User sees the actual problem, not a sea of red logs.
2. **Global** `window.onerror` + `unhandledrejection` → `atlasLog('error', ...)` → `#error-log` panel.
3. **Per-renderer** `safeRender(name, fn)` → one render failure logs and is skipped; others run.
4. **Per-event-listener** `on(id, ev, handler)` → missing element logs a warning instead of throwing.
5. **Per-DOM-read** `$(id)` → missing element returns null + logs.
6. **Per-MCP-call** `callTool()` returns `{__error: msg}` instead of rejecting; runtime-missing case short-circuits with a clear message.

The `#error-log` panel (collapsed by default, page bottom) is the single observability surface. Counter shows `<total> · <N> err · <M> warn`.

---

## Patterns & conventions

| Pattern | Used for |
|---|---|
| **innerHTML string concatenation** | Every renderer. No virtual DOM, no template engine. |
| **`safeText(s)`** | HTML-escape any user/MCP-derived string before interpolating. **Always use it for `utm_*`, `referrer_name`, post content, file names, query labels.** |
| **`fmtInt`, `fmtMoney`, `fmtPctRaw`, `fmtPct`** | All numeric formatting. They handle null/NaN. |
| **`cvrClass(rate)`** | Returns `cvr-good` / `cvr-mid` / `cvr-low` for the colored CVR pill. |
| **Sort state in `STATE.sort.<table>`** | Header click → `onSortClick` mutates `{col, dir}` → re-render. |
| **`ChartManager.render(canvasId, configFn, updateFn)`** | Always. Don't `new Chart()` directly anywhere — the cache reconcile is what keeps stale instances from leaking. |
| **localStorage prefix `cairo_`** | Every persisted key. Set by `ATLAS_CONFIG.storagePrefix` — change when forking. |

---

## Known sharp edges

- **CVR units**. `STATE.cache.kpi.cvr` is a fraction (0–1); per-row `cvr` in tables is already a percent (0–100). Don't double-multiply. Should be standardized to "always percent" in `buildCache`.
- **Metricool 24h lag**. When range is "today only", `getMetricoolRange()` quietly switches to yesterday and labels it `fallback: true`. The Posts panel UX explains it; renderers must respect the `fallback` flag.
- **Previous-period UTC drift**. `previousPeriod()` parses ranges as UTC (`'YYYY-MM-DDT00:00:00Z'`); for shop-local ranges crossing UTC midnight there can be a one-day cosmetic drift in the "compared vs" label.
- **`fetchAll()` bypasses `atlasGate`**. The 17-call burst doesn't go through the rate-limited `atlasApiCall`. Should be routed through it for stricter MCP hosts.
- **`window.cowork` not present**. Boot probe banners this now. Without the runtime: every fetch and chat fails. Confirm by reading the top banner before hunting in `#error-log`.
- **MCP UUIDs are per-Cowork-project**. The committed UUIDs are the original author's. On fork, they almost certainly need to be replaced — see `STACK.md` → "Swapping MCP UUIDs". This is the #1 cold-clone failure mode.
- **`.settings-panel` CSS exists, no markup**. Either dead CSS or planned feature — flagged for cleanup.
