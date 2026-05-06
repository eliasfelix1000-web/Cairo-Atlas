# Stack

What CĀIRO Atlas is built out of and how the pieces fit. Use this when you're swapping libraries, swapping MCP servers, or porting the artifact to a different environment.

---

## Deployment shape

A single HTML file (`index.html`) that runs as a Claude.ai live artifact inside a Cowork project. No build step, no bundler, no server. The file is opened by the Cowork artifact runtime, which:

- Parses the `<script type="application/json" id="cowork-artifact-meta">` block at the top to know which MCP servers to wire.
- Exposes `window.cowork.callMcpTool(name, args)` for MCP tool calls.
- Exposes `window.cowork.askClaude(prompt, attachments)` for in-artifact AI completions.

Anything that breaks the single-file shape (split into modules, npm install, server-side build) breaks the artifact contract. **Keep `index.html` as one file.**

The local `index.html` on disk is a backup / diff target. The runtime source-of-truth is the live artifact inside Claude.ai. Push commits → Cowork doesn't auto-pick-them-up — you re-paste into the live artifact (or ask Claude inside the Cowork task to apply the diff).

---

## Runtime dependencies

| Layer | Choice | Why |
|---|---|---|
| Charts | **Chart.js 4.5.0** UMD via jsDelivr (with SRI hash) | Only library shipped. Doughnut + bar + line + combo are native. Single file, no build. |
| Framework | **None — vanilla JS in strict mode** | Every renderer is a function that writes `innerHTML`. Stays portable across artifact runtimes. |
| Styling | **Custom CSS with CSS variables** | Dark warm-parchment theme: `--bg`, `--panel`, `--accent`, `--p-*` (platform brand), `--m-*` (medium semantic), `--pos`/`--neg`/`--warn`. ~1200 lines of CSS in `<style>`. |
| Typography | System stack (`-apple-system`, Inter, Segoe UI, SF Pro Text) | No webfont round-trip. Tabular-nums everywhere numbers appear. |
| Icons | Inline SVG | No icon font. |
| State persistence | `localStorage`, prefix `cairo_` | Refresh interval, paused flag, range, custom dates, layout (drag order), chat history (last 50), chat sizes, cached suggestions. Prefix is `ATLAS_CONFIG.storagePrefix` — rename when forking. |

**Cowork runtime contract** — the runtime provides:

- `window.cowork.callMcpTool(name, args)` → MCP tool invocation, returns whatever the tool returns (often wrapped in `{content: [{text}]}` — see `unwrapResponse()` in code).
- `window.cowork.askClaude(prompt, attachments)` → AI completion call. Used by the chat bubble and dynamic suggestion generator.

These are **artifact-runtime globals**. The dashboard cannot run as a static page in a browser without them — every data fetch and the chat will fail (and the new boot probe `probeCoworkRuntime()` will banner the failure visibly). For local UI iteration on the markup/CSS, mock `window.cowork` on `window` before `index.html` loads.

---

## MCP servers

### Servers used by the example (Atlas)

Declared in `cowork-artifact-meta` at the top of `index.html` and consumed via `ATLAS_CONFIG.mcp.*` (frozen object near the top of the script, `window.ATLAS_CONFIG`).

| Server | Tool name pattern | Purpose | Connector |
|---|---|---|---|
| Shopify | `mcp__…__run-analytics-query` | All sessions / orders / UTM / funnel queries via ShopifyQL. ~10 distinct queries per refresh + 5 prev-period. | Official `claude.ai` Shopify connector (OAuth) |
| Metricool | `mcp__mcp-metricool__get_instagram_reels` | IG Reel metrics (reach, plays, engagement, time). 24h lag. | Custom HTTP MCP — see "Adding Metricool" below |
| Metricool | `mcp__mcp-metricool__get_tiktok_videos` | TikTok video metrics. Same lag. | Same |
| Motion | `mcp__…__get_creative_insights` | Last-30d Meta paid creative table for grading. | Official `claude.ai` Motion Creative Analytics connector |
| Motion | `mcp__…__get_creative_summary` | Per-creative AI reasoning. | Same |
| Motion | `mcp__…__get_creative_transcript` | Hook + script transcript for the slide. | Same |

Brand-specific constants in `ATLAS_CONFIG`:

- `metricoolBlogId: 6200400` — the original author's Metricool blog id. **Replace on fork.** Find yours in your Metricool admin URL.
- `shopTz: 'Europe/Stockholm'` — affects all date bucketing. Replace with your store's timezone.

### Adding Metricool (custom HTTP MCP)

Metricool is **not** an official `claude.ai` connector. Add it as a custom MCP:

- **Claude.ai web** → Settings → Connectors → Add custom connector → HTTP transport → URL `https://ai.metricool.com/mcp` → authenticate via Metricool OAuth.
- **Claude Code** (alternate path, command-line projects) → `claude mcp add --transport http metricool https://ai.metricool.com/mcp`.

Reference: [Metricool MCP install docs](https://help.metricool.com/en/article/how-to-connect-metricools-mcp-1364s63/) · [`metricool/mcp-metricool` on GitHub](https://github.com/metricool/mcp-metricool).

### Swapping MCP UUIDs (on fork)

The tool names in `ATLAS_CONFIG` look like `mcp__<uuid>__<tool>`. The UUID prefix is **per-Cowork-project** for some MCP servers (Shopify, Motion at minimum) — it is the registered server-instance id, not a canonical name. When you create a fresh Cowork project and connect your own MCP servers, the UUID prefixes will differ from the ones committed in this repo.

**Procedure on first run:**

1. Open the live artifact's browser devtools console.
2. Inspect `window.cowork` and `window.ATLAS_CONFIG.mcp` — note what's *attempted* vs what would *resolve* on your account.
3. In the parent Cowork task, ask Claude: *"List the MCP tools available to this artifact."* (Or call any `window.cowork` introspection method if one exists — confirm by reading `Object.keys(window.cowork)`.)
4. Edit `index.html` → search for `ATLAS_CONFIG = Object.freeze` → replace each `mcp.*` string with the matching tool name from your environment.
5. Also update the `mcpTools` array in the `cowork-artifact-meta` JSON block at the very top of the file (the runtime parses this to wire connectors).
6. Re-paste / re-deploy the artifact. The boot probe will banner any tool name that doesn't resolve.

If you see `__error: window.cowork.callMcpTool unavailable` in the logs, you're not running inside a Cowork iframe. If you see `__error: tool not found` for every call, it's a UUID mismatch — back to step 3.

### Alternatives by category

| Category | Default | Alternatives |
|---|---|---|
| Ecommerce analytics | Shopify | WooCommerce / BigCommerce custom MCP; PostgreSQL MCP if you ETL into a warehouse |
| Social analytics | Metricool | Direct Meta Graph API MCP, TikTok API MCP, Sprout Social custom MCP |
| Paid creative analytics | Motion | Direct Meta Marketing API MCP; Triple Whale / Northbeam custom MCP |
| Email | (not wired in example) | Klaviyo official `claude.ai` connector |
| Chat AI | `window.cowork.askClaude` (Cowork-native) | Switch to direct Anthropic API calls if porting outside Cowork |

When swapping, keep the call shape identical — every fetch flows through `callTool(name, args)` which is a no-op rate-limited + error-logged wrapper. The renderers consume `STATE.cache.*` slices that are independent of source.

---

## Browser support

Modern evergreen browsers only. Uses optional chaining, nullish coalescing, `async/await`, `Object.entries`, `Intl.DateTimeFormat` with timezone, CSS custom properties, `backdrop-filter`. No IE, no transpilation safety net. Target whatever Chromium ships in current Claude Desktop / current Chrome stable.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Top banner: "Cowork runtime not detected" | Opened `index.html` directly in a browser, not as a Cowork artifact | Re-deploy via Cowork (README §"Run it cold") |
| Top banner: "Chart.js failed to load" | jsDelivr blocked or SRI mismatch | Vendor Chart.js inline (paste minified UMD into a `<script>` block above the dashboard `<script>`); remove the CDN tag |
| All MCP calls error in `#error-log` | UUID mismatch | STACK §"Swapping MCP UUIDs" |
| Shopify works, Metricool fails | Custom MCP not connected on this Cowork project | Settings → Connectors → Add custom connector (URL above) |
| `auth_error` / `401` in logs for one server | OAuth token expired in Cowork | Settings → Connectors → that server → reconnect |
| Layout drags into a mess | localStorage corrupted | Click "Reset layout" in topbar (clears `cairo_atlas_layout_v1`) |
| Numbers off by hours on edges | `shopTz` wrong for your store | Change `ATLAS_CONFIG.shopTz` |
