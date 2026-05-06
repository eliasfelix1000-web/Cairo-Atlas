# CĀIRO Atlas

Live attribution and organic-content funnel dashboard for a Shopify-powered DTC brand. Single-file HTML Claude Cowork artifact — open `index.html` in the Cowork artifact runtime and it boots.

> **Status:** v0.1 — patched. Initial baseline commit is the load-fix only; the OOP refactor and full docs land in the commits that follow.

## What it does

Auto-refreshes (15s / 30s / 60s / 2m) and unifies three telemetry streams onto one screen:

- **Shopify analytics** (via `run-analytics-query`) — sessions, orders, revenue, UTM source/medium/campaign, referrer source/name, funnel stages (sessions → cart → checkout → order).
- **Metricool** (via `get_instagram_reels` + `get_tiktok_videos`) — IG Reel + TT video reach and engagement; lags ~24h, so when the range is "today" it auto-falls-back to yesterday and labels it.
- **Motion Creative Analytics** (manual load) — picks 3 paid Meta creatives from the last 30 days (Best / Underrated / Worst by ROAS + significance) and runs the AI reasoning analysis on each.

The intended user is a growth or performance marketer (or founder) at a small e-commerce brand who needs to answer "where is traffic coming from, what content drove it, what should I scale or kill" without bouncing between Shopify Admin, Metricool, and Motion.

## Sections

1. **At a glance** — five top-line KPIs (sessions, UTM-tagged share, orders, revenue, CVR) with vs-prev-period delta arrows.
2. **Volume over the period** — hourly/daily sessions+orders combo chart.
3. **Channel breakdown** — Sidekick-style channel buckets, sortable. Variant granularity: `utm_source=igstory` shows as *"Instagram organic — Story"*, `igreel` as *"— Reel"*, etc.
4. **Content → Site funnel** — content stats from Metricool joined with the Shopify organic UTM funnel: KPIs, per-platform table, reach donut, post bars, daily organic chart.
5. **Organic content** — IG Reel + TT video cards.
6. **Creative analysis** — Motion API slideshow grading 3 paid creatives.
7. **UTM source & medium** — twin sortable drill-down tables.
8. **Top campaigns** — top-15 source/medium/campaign sortable table.
9. **Source × medium matrix** — heatmap + UTM coverage.
10. **Funnel snapshot** — sessions → cart → checkout → order drop-off.
11. **Logs** — collapsible error/info log fed by `atlasLog()`.
12. **Cairo AI chat** — floating glass-effect FAB, sends the current data snapshot to Claude.

## Stack

- **Single-file HTML** — Claude Cowork artifact. Top of file holds the `cowork-artifact-meta` JSON declaring required MCP tools.
- **Chart.js 4.5.0** UMD via jsDelivr (only runtime dependency).
- **Vanilla JS** in strict mode. No build step. No framework.
- **CSS** custom-properties driven dark theme.
- **MCP tools** consumed via `window.cowork.callMcpTool(name, args)`.
- **AI chat** uses `window.cowork.askClaude(prompt, attachments)`.

See [`STACK.md`](STACK.md) for the full dependency map and [`ARCHITECTURE.md`](ARCHITECTURE.md) for module/class layout.

## Running

This is a Cowork artifact, not a static site:

1. Open `index.html` in a Claude Cowork environment that has the declared MCP servers connected:
   - `Shopify` — for `run-analytics-query`
   - `mcp-metricool` — for `get_instagram_reels`, `get_tiktok_videos`
   - `Motion Creative Analytics` — for `get_creative_insights`, `get_creative_summary`, `get_creative_transcript`
2. The dashboard boots, calls all queries on the current date range, and starts the auto-refresh loop.
3. Pause / change interval / change date range from the topbar.

If you open the file directly in a browser without the Cowork runtime, every MCP call will fail and the Logs panel will fill with errors — but the UI shell itself will render.

## Repo

- `index.html` — the artifact. Everything lives here.
- `versions/` — historical snapshots from the Cowork iteration history.
- `STACK.md` — dependencies and how the pieces fit.
- `ARCHITECTURE.md` — module/class layout and data flow.
- `CLAUDE.md` — how a future Claude session should navigate the project.
- `CHANGELOG.md` — every patch, in order.

## License

Private project. All rights reserved.
