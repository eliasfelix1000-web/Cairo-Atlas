# Architecture

The actual code architecture inside `index.html`. Read this before touching the script.

## One-screen mental model

```
                ┌────────────────────────────────────────────┐
                │                window.cowork               │
                │  callMcpTool(name, args)                   │
                │  askClaude(prompt, attachments)            │
                └──────────┬───────────────────┬─────────────┘
                           │                   │
                  (MCP data calls)      (chat completions)
                           │                   │
                ┌──────────▼───────┐  ┌────────▼────────┐
                │   ATLAS / api    │  │   CHAT / chat   │
                │  rate-limit, log │  │  history, send  │
                └──────────┬───────┘  └────────┬────────┘
                           │                   │
                ┌──────────▼───────────────────▼──────────┐
                │              fetchAll()                 │
                │   17 parallel MCP calls per refresh     │
                └──────────┬──────────────────────────────┘
                           │
                ┌──────────▼──────────────────────────────┐
                │            buildCache(data)              │
                │  pure aggregator → STATE.cache.{...}     │
                └──────────┬──────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
       ┌──────▼──────┐           ┌──────▼──────┐
       │ ChartManager│           │  Renderers  │
       │  (STATE.    │           │  (KPIs,     │
       │   charts.*) │           │   tables,   │
       │             │           │   funnel,   │
       │             │           │   content)  │
       └─────────────┘           └─────────────┘
                                        │
                                  ┌─────▼──────┐
                                  │    DOM     │
                                  │  (#kpi-*,  │
                                  │  #channel- │
                                  │  table…)   │
                                  └────────────┘
```

The whole thing runs on a `setTimeout` loop driven by `STATE.intervalSec`.

## Modules / classes

The script is procedural; the OOP **facade** at `window.CairoAtlas` (end of script) is a thin class layer over the existing singletons. Each "module" below corresponds to one of those classes plus the singleton it owns.

### `AtlasApi` → `ATLAS` singleton (~line 1463)
Rate-limited MCP wrapper. Owns:
- `atlasGate()` — yields until inflight slot + global throttle (80ms floor) clear.
- `atlasApiCall(toolName, args, opts)` — full pipeline: gate → race against a 16s timeout → log slow/error → release.
- `atlasLog(level, source, message, details)` — append to `ATLAS.logs` (capped at 60), trigger `renderErrorLog()` to update the `#error-log` panel, mirror `error`/`warn` to console.

`ATLAS.api.counters` exposes `{total, errors, slow}` for sanity checks from devtools.

`window.CairoAtlas.api` proxies this for navigability.

### `Dashboard` → `STATE` singleton (~line 1577)
`STATE` carries all dashboard state:

```js
STATE = {
  range, custom,                     // active date range (preset id + optional custom dates)
  intervalSec, paused, timer,        // refresh loop control
  lastRefreshedAt,                   // Date — drives the freshness pill
  charts: {volume, content, donut, postBars},   // Chart.js instances (destroyed/recreated on refresh)
  sort: {channel, source, medium, campaigns},   // {col, dir} per table
  cache: { ... slices },             // see "Cache shape" below
  insightHistory                     // CA history (currently unused outside CA)
}
```

`Dashboard` methods proxy `refresh()`, `scheduleNext()`, `setRange()`, `setIntervalSec()`, `setPaused()`, `manualRefresh()`, `fetchAll()`, `buildCache()`.

### `ChannelClassifier` (functions, no singleton, ~line 1700)
Pure functions:
- `bucketSessionChannel(src, med, refSrc, refName)` — sessions side: uses `referrer_source` + `referrer_name`.
- `bucketOrderChannel(src, med, refChan)` — orders side: uses Shopify's `referring_channel`.
- `variantSuffix(src)` — turns `igstory`/`igreel`/`igbio`/`tt*`/`fb*` into the variant tail.

Order of precedence: Meta paid → IG/TT/FB organic with variant → Klaviyo email → Google organic → social referrer fallback → Direct → Other.

### `DataService` (functions, ~line 1840)
- `fetchAll()` — fires 10 Shopify queries + 2 Metricool calls + 5 prev-period Shopify + 2 prev-period Metricool in parallel via `Promise.all`. Returns `{key: rawResult, ...}`. **Bypasses `atlasGate` for now** — the gate only protects calls routed through `atlasApiCall`. Each tool call is wrapped in `callTool()` which catches and returns `{__error: msg}` so the promise never rejects.
- `buildCache(data)` — pure transform. Keys produced under `STATE.cache`:

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

### `ChartManager` → `STATE.charts` (no class, accessed as `window.CairoAtlas.charts`)
Pattern is identical for every chart:

```js
const ctx = canvas.getContext('2d');
if (STATE.charts.X) { STATE.charts.X.destroy(); STATE.charts.X = null; }
STATE.charts.X = new Chart(ctx, config);
```

`renderReachDonut` and `renderPostBars` lazily re-inject the `<canvas>` element if the wrap was emptied. `renderContentChart` and `renderVolumeChart` do not — if their canvas was replaced by a skeleton/empty state, they'll throw on `.getContext` (caught now by `safeRender`).

### Renderers (functions, ~line 2200–3000)
One per UI section. Conventions:

- `renderKpis`, `renderChannelTable`, `renderSourceTable`, `renderMediumTable`, `renderCampaignsTable`, `renderMatrix`, `renderCoverage`, `renderFunnel`, `renderContentFunnel`, `renderVolumeChart`, `renderReachDonut`, `renderPostBars`, `renderContentChart`, `renderPosts`.
- Each reads from `STATE.cache.<slice>` and writes into a fixed DOM id.
- Sortable tables share `thHTML(col, label, sortKey, sort, align)` for header rendering and `sortRows(rows, col, dir)` for ordering.
- Bars in tables are CSS divs (`.bar-track > .bar-fill`) sized by `(value / max) * 100%`.
- Channel tables append a `totalrow`; campaigns table does not (intentional or oversight).

After the refactor, every renderer is invoked through `safeRender('renderName', fn)` in `refresh()` — one failure logs to the panel and the rest of the pass continues.

### Creative Analysis (CA) → `CA` singleton (~line 2802)
Heavier subsystem (7+ API calls per analysis), manually triggered by the `#ca-load-btn` button. Owns its own state (`CA.currentSlide`, slide content, transcripts), its own renderers (`caRender`, `caRenderSlide`, `caRenderReasoning`), and its own progress UI. Picks Best/Underrated/Worst from Motion's last-30d Meta spend table.

### `DateRangePicker` → `DR` singleton (~line 3500)
Custom calendar (Mon-first, range select, future-day disable, prev/next month nav, preset buttons). Apply writes `STATE.range = 'custom'` + `STATE.custom = {from, to}`, persists to localStorage, and triggers `refresh()`.

### `ChatBubble` → `CHAT` singleton (~line 3850)
Floating glass-FAB Cairo AI chat. Compresses `STATE.cache` into a slim payload (`compressCacheForChat()`), calls `window.cowork.askClaude(prompt, [data])`, renders the conversation, generates 3 dynamic suggestion prompts in the background (`ensureSuggestions()`). Persists history (last 50), text size, window size, and suggestions to localStorage.

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
            ├─ safeRender('renderVolumeChart', …)
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

## Error handling philosophy (post-refactor)

> "Fail loud, log to panel, never freeze."

1. **Global** `window.onerror` + `unhandledrejection` → `atlasLog('error', ...)` → visible in `#error-log` panel.
2. **Per-renderer** `safeRender(name, fn)` → one render failure logs and is skipped; others run.
3. **Per-event-listener** `on(id, ev, handler)` → missing element logs a warning instead of throwing.
4. **Per-DOM-read** `$(id)` → missing element returns null + logs, so caller can guard.
5. **Per-MCP-call** `callTool()` returns `{__error: msg}` instead of rejecting; `atlasApiCall()` logs slow + error explicitly.

The Logs panel (collapsed by default at the bottom of the page) is the single observability surface. If something looks off, open the Logs first. The badge counter shows `<total> · <N> err · <M> warn`.

## Patterns & conventions

| Pattern | Used for |
|---|---|
| **innerHTML string concatenation** | Every renderer. No virtual DOM, no template engine. |
| **`safeText(s)`** | HTML-escape any user/MCP-derived string before interpolating. **Always use it for `utm_*`, `referrer_name`, post content, file names, query labels.** |
| **`fmtInt`, `fmtMoney`, `fmtPctRaw`, `fmtPct`** | All numeric formatting. Don't roll your own — they handle null/NaN. |
| **`cvrClass(rate)`** | Returns `cvr-good` / `cvr-mid` / `cvr-low` for the colored CVR pill. |
| **Sort state in `STATE.sort.<table>`** | Header click → `onSortClick` mutates `{col, dir}` → re-render. |
| **Chart destroy-then-recreate** | Always: `if (STATE.charts.X) STATE.charts.X.destroy()` before `new Chart(...)`. |
| **localStorage prefix** | Every persisted key is prefixed `cairo_` — never collide with other artifacts. |

## Known sharp edges

- **CVR units**: `STATE.cache.kpi.cvr` is a fraction (0–1); per-row `cvr` in tables is already a percent (0–100). Don't double-multiply.
- **Metricool 24h lag**: when range is "today only", `getMetricoolRange()` quietly switches to yesterday and labels it `fallback: true`. The Posts panel UX explains it; renderers must respect the `fallback` flag.
- **Previous period**: `previousPeriod()` parses ranges as UTC (`'YYYY-MM-DDT00:00:00Z'`); for shop-local ranges crossing UTC midnight there can be a one-day cosmetic drift in the "compared vs" label.
- **Empty `STATE.cache.*`**: most renderers early-return on missing slice but leave stale DOM. After the refactor, errors are logged but the UX impact is the same.
- **`window.cowork` not present**: every fetch and chat call will fail. With the new global error handler, you'll see a flood of errors in the Logs panel — that's the symptom of running outside a Cowork artifact runtime.
