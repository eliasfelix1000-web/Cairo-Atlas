# Stack

What CĀIRO Atlas is built out of and how the pieces fit. If you're touching code, this is the map of "what library does what".

## Deployment shape

**A single HTML file (`index.html`) that runs as a Claude Cowork artifact.** No build step, no bundler, no server. The file is opened by the Cowork artifact runtime, which:

- Parses the `<script type="application/json" id="cowork-artifact-meta">` block at the top to know which MCP servers to connect.
- Exposes `window.cowork.callMcpTool(name, args)` for MCP tool calls.
- Exposes `window.cowork.askClaude(prompt, attachments)` for the in-app AI chat.

Anything that breaks the single-file shape (split into modules, npm install, server-side build) will break the artifact contract. **Do not refactor toward multi-file.** Keep the surgical class facade and the inline `<style>`/`<script>` blocks where they are.

## Runtime dependencies

| Layer | Choice | Why |
|---|---|---|
| Charts | **Chart.js 4.5.0** UMD via jsDelivr (with SRI hash) | Only library we ship. Doughnut + bar + line + combo line/scatter are all native. Single file, no build. |
| Framework | **None — vanilla JS in strict mode** | Every renderer is a function that writes `innerHTML`. Stays portable across artifact runtimes. |
| Styling | **Custom CSS with CSS variables** | Dark warm-parchment theme defined as `--bg`, `--panel`, `--accent`, `--p-*` (platform brand), `--m-*` (medium semantic), `--pos`/`--neg`/`--warn`. ~1100 lines of CSS in `<style>`. |
| Typography | System stack (`-apple-system`, Inter, Segoe UI, SF Pro Text) | No webfont round-trips. Tabular-nums everywhere numbers appear. |
| Icons | Inline SVG | No icon font. |
| State persistence | `localStorage` keys `cairo_utm_*` and `cairo_chat*` | Refresh interval, paused flag, range, custom dates, chat history (last 50), chat sizes, suggestions cache. |

The `claudeRuntime` (Cowork) provides:
- `window.cowork.callMcpTool(name, args)` → MCP tool invocation
- `window.cowork.askClaude(prompt, attachments)` → AI completion call

These are **artifact-runtime globals**. The dashboard cannot run as a static page in a browser without them — every data fetch and the chat will fail (and now log loudly to the `#error-log` panel). For local UI iteration, mock these on `window` before `index.html` loads.

## Data sources (MCP)

Declared in `cowork-artifact-meta` at the top of `index.html` and consumed via the `TOOL_*` constants near line 1500.

| Source | Tool name | Purpose |
|---|---|---|
| Shopify | `mcp__…__run-analytics-query` | All sessions/orders/UTM/funnel queries via ShopifyQL. One tool covers ~10 distinct queries per refresh + 5 prev-period queries. |
| Metricool | `mcp__mcp-metricool__get_instagram_reels` | IG Reel metrics (reach, plays, engagement, time). Lags ~24h. |
| Metricool | `mcp__mcp-metricool__get_tiktok_videos` | TikTok video metrics. Same lag. |
| Motion (manual load) | `mcp__…__get_creative_insights` | Last-30d Meta paid creative table for grading. |
| Motion (manual load) | `mcp__…__get_creative_summary` | Per-creative AI reasoning. |
| Motion (manual load) | `mcp__…__get_creative_transcript` | Hook + script transcript for the slide. |

Metricool blog id is hardcoded: `METRICOOL_BLOG_ID = 6200400`. Shopify timezone is `Europe/Stockholm`.

The Cowork environment may register the Shopify/Drive/Motion MCP servers under different IDs than the UUIDs hardcoded in `TOOL_*` constants — if every Shopify call is failing in the log panel, the tool name needs updating to match the live registration.

## Architecture entry points

If you only have time to read 50 lines of code, read these:

| What | Where | Why |
|---|---|---|
| Top-of-script ToC + safety helpers | `index.html` ~1387–1460 | Comment block listing all 12 sections in line order; `$()`, `on()`, `safeRender()` helpers + global `window.onerror`. |
| `ATLAS` orchestrator + `atlasLog` | ~1463–1540 | Rate-limited MCP wrapper, log buffer (max 60 entries), used by every data fetch. |
| `STATE` singleton | ~1577–1610 | Range, interval, paused, cache, chart instances, sort state. |
| `bucketSessionChannel` / `bucketOrderChannel` | ~1690–1730 | UTM → channel bucketing (Meta paid / IG/TT/FB organic with Story/Reel/Bio variants / Klaviyo email / Google organic / Direct / Other). |
| `fetchAll()` | ~1840 | One-shot data pull — 17 parallel MCP calls. |
| `buildCache(data)` | ~1903 | Pure aggregator — turns raw rows into the renderer-friendly `STATE.cache.*` slices. |
| `refresh()` | ~3490 | Top of the dashboard loop. Each renderer wrapped in `safeRender()` so one failure doesn't kill the rest. |
| Boot IIFE + `window.CairoAtlas` facade | end of script (~4150) | Class wrappers + init steps. Devtools entry point. |

## What lives where (file map)

```
index.html                  the artifact — everything is here
├─ <script id="cowork-artifact-meta">     declares MCP tool requirements
├─ <script src="…chart.js">                only CDN dependency
├─ <style>                                  ~1100 lines of dark theme + components
├─ <body>                                   ~250 lines of static markup with element IDs
└─ <script>                                 ~2700 lines of dashboard JS
   1. error handling + DOM helpers
   2. ATLAS api orchestrator + logger
   3. tool constants
   4. STATE
   5. classifiers (utm → channel)
   6. fetchAll + buildCache
   7. renderers (kpis, tables, charts, funnel, content, posts)
   8. CA — Creative Analysis
   9. refresh / scheduleNext / setRange
  10. DR — date range picker
  11. CHAT — Cairo AI chat bubble
  12. boot — class facade + init

versions/                    historical snapshots from Cowork iterations
README.md                    elevator pitch + how to open
STACK.md                     this file
ARCHITECTURE.md              module/class layout + data flow
CLAUDE.md                    onboarding for the next Claude session
CHANGELOG.md                 every patch
```

## Browser support

Modern evergreen browsers only. Uses optional chaining (`?.`), nullish coalescing (`??`), `async/await`, template literals, `for…of`, `Object.entries`, `Intl.DateTimeFormat` with timezone, CSS custom properties, `backdrop-filter`. No IE, no transpilation safety net.
