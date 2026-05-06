# Changelog

Every patch on `main`, newest first. Each entry should answer "what + why" — file paths and line numbers belong in the diff, not here.

Format inspired by [Keep a Changelog](https://keepachangelog.com/) but loosened: pre-1.0, no SemVer yet.

---

## 2026-05-06 (late) — Open-source restructure: Cairo AI Cowork artifact starter

The repo positioning shifted from "G's private dashboard fix log" to **Cairo AI's open-source Cowork artifact starter**, with CĀIRO Atlas as the example app. The first cold-clone (G pulling onto a fresh laptop and standing it up in Cowork) would have hit ~20 silent failure modes — most fatally, hardcoded MCP UUIDs that don't match a fresh Cowork project. This release closes those gaps in code, docs, and repo hygiene so the cold-clone path is rust-level confident.

### Added
- **`ATLAS_CONFIG`** — frozen object near the top of the dashboard script. One place to edit when forking: MCP tool names (Shopify / Metricool IG+TT / Motion CA insights+summary+transcript), `metricoolBlogId`, `shopTz`, `storagePrefix`. Existing `TOOL_SHOPIFY` / `TOOL_IG_REELS` / `TOOL_TT_VIDEOS` / `METRICOOL_BLOG_ID` / `SHOP_TZ` are now thin aliases off the config so no call site changes. Exposed as `window.ATLAS_CONFIG`.
- **`__atlasBanner(id, level, html)`** — fixed-position top-of-page banners injected directly into `<body>`. For things the user must see even if the rest of the dashboard never paints (missing Cowork runtime, Chart.js CDN failure). Stacks vertically, dismissable, idempotent.
- **`probeCoworkRuntime()`** — runs from the init IIFE. Banners visible breakage when `window.cowork.callMcpTool` or `Chart` is missing instead of letting silent failure look like "blank dashboard forever".
- **`callTool` runtime guard** — short-circuits with a friendly `__error` payload when `window.cowork` is absent, instead of throwing on `undefined`.
- **Chart.js `onerror` hook** — sets `window.__atlasChartLoadFailed` so the boot probe can banner CDN failures without a page crash.
- **`.gitattributes`** — forces LF on every text file. Prevents the JSON `cowork-artifact-meta` block from breaking on Windows ↔ Mac re-clones (autocrlf can leave a trailing `\r` that some runtimes refuse to parse).
- **`.editorconfig`** — indent / charset / line-ending alignment without language-specific tooling install.
- **`LICENSE`** — MIT. Repo flips from "private, all rights reserved" to open-source-by-default.

### Changed
- **README.md** rewritten as a public open-source landing. 30-second pitch, demo screenshot, "Run it cold" numbered runbook (clone → Claude Code → Cowork upload → MCP connect → UUID swap → live), required-MCP-servers table with custom-Metricool path, diagnostic command list, alternatives, MIT call-out, Cairo AI credit line.
- **STACK.md** rewritten with explicit "Swapping MCP UUIDs" procedure (the #1 cold-clone failure), a custom-Metricool install section (pointing at `https://ai.metricool.com/mcp` for both Claude.ai web and Claude Code), an "Alternatives by category" table, and a top-level Troubleshooting table mapping symptoms → fixes.
- **ARCHITECTURE.md** rewritten to reflect the actual procedural / module-singleton structure. The `window.CairoAtlas` class facade is recast as a one-paragraph devtools navigability aid at the bottom of the doc — not the spine. Module map now lists `ATLAS`, `STATE`, `ChartManager`, `LayoutManager`, `CA`, `DR`, `CHAT` as the singletons they are.
- **CLAUDE.md** restructured into a two-audience contract. Generic-clone subcontract (don't assume auto-memory, don't assume preferences, walk the user through MCP UUID swap on first task) is the default. G-specific content (CĀIRO business context, four-hats role, daily-log, auto-memory path) is fenced under a "G-specific context" section that external clones can ignore.

### Removed
- Internal-handoff voice across the docs ("what I (Claude Opus 4.7) learned arriving", "the next Claude session", "OOP refactor" framing as architecture). Substance migrated where it still belongs: the load-fix history stayed in the prior CHANGELOG entry, the "what's still risky" list moved to ARCHITECTURE.md → "Known sharp edges" with neutral voice, the "what I added" went into the prior CHANGELOG entry's prose.
- README's "Status: v0.1 — patched" + "Private project. All rights reserved." stale framing.
- STACK's "surgical class facade" phrasing.

### Open questions still on G
- Whether `https://github.com/eliasfelix1000-web/Cairo-Atlas.git` is set to public yet. Hardcoded `metricoolBlogId` and Cowork-project-bound MCP UUIDs effectively ship author-specific identifiers if it is.
- Whether `.settings-panel` CSS without markup is dead code or a planned feature (still flagged in ARCHITECTURE → "Known sharp edges").
- Whether Cowork's CSP allows `cdn.jsdelivr.net` in all environments. If not, vendor Chart.js inline (procedure documented in STACK → "Troubleshooting").

---

## 2026-05-06 (PM) — Layout architecture refactor + drag-and-drop

The dashboard was accreting features faster than the base could absorb them — every new metric required edits in three places (HTML element, CSS grid column count, renderer wiring), and rudimentary bugs (chart cache against detached canvas, sub-line clipped tooltips) kept reappearing. This release moves the foundation from imperative DOM mutation to slot discovery + lifecycle-managed charts.

### Added
- **`ChartManager`** — centralized Chart.js lifecycle with self-healing cache. Every render verifies the cached instance is still bound to the live canvas; mismatch → destroy + recreate. The class of bug we hit twice (donut + post-bars empty-state path leaking a stale chart reference) can no longer recur. Also has a `reconcile()` step that runs after layout swaps to flush charts whose canvases left the DOM.
- **`LayoutManager`** — slot-discovery layout engine. Every draggable element on the page declares `data-slot="..."` + `data-slot-group="..."`; on boot the manager scans the DOM, applies any saved order from localStorage, and wires drag handlers per slot. Adding a new metric tile is now a one-HTML-edit operation — the layout system picks it up automatically including drag support and persistence.
- **Drag-and-drop UX** — six-dot drag handle in the corner of every KPI tile and panel, fades in on hover, fully transparent otherwise. Dragging a slot highlights the valid drop targets (peers in the same group) and dims everything else. Drops swap positions and persist to localStorage. Touch devices: handles auto-hide via `@media (hover: none)`.
- **Reset layout button** — topbar control that confirms then clears the saved layout and reloads, so a messy drag session can be undone in one click.
- **Six draggable groups** wired up: at-a-glance KPIs (5 tiles), Content → Site funnel KPIs (6 tiles), funnel snapshot KPIs (4 tiles), Reach donut + Per-post engagement panels, UTM source + medium panels, Session matrix + UTM coverage panels.
- **`views_per_sess`** efficiency metric on the Total reach/views KPI: `total_reach / organic_sessions` — how many organic views to produce one click-through. Native title-attr tooltip (kpi-sub has overflow:hidden for line-clamping, which clips popover-style tooltips).

### Changed
- All four chart renderers (`renderReachDonut`, `renderPostBars`, `renderContentChart`, `renderVolumeChart`) migrated to use `ChartManager.render(canvasId, configFn, updateFn)`. Removed the manual cache-then-update-or-create branches; halved their line counts. Empty-state paths now call `ChartManager.destroy(id)` instead of manually nulling out cached instances.
- `STATE.charts` removed entirely — chart instances now live exclusively in `ChartManager._charts`.
- `.kpi { overflow: hidden }` removed so info-icon tooltips can escape upward (was clipping every KPI tooltip — silent bug).
- `.kpi-sub` line-clamp bumped 2 → 3 lines (max-height 2.8em → 4.2em) to fit the now-3-clause sub-line on the reach KPI without truncation.
- `.panel[data-slot] > h3 { padding-right: 24px }` so the right-aligned `.hint` doesn't get covered by the corner drag handle on hover.

### Architecture notes for next session
- The slot discovery model: `LayoutManager._scanGroups()` finds all `[data-slot][data-slot-group]` elements and groups them by `groupId @ parentSelector`. Slots can only swap with peers in the same group AND with the same parent. To add a new draggable peer, just put `<div ... data-slot="some-id" data-slot-group="existing-group-id">` in the HTML — no JS change needed.
- The chart self-heal: if you ever introduce a new code path that replaces a canvas (e.g. a new empty-state UI), ChartManager handles it transparently. You can call `ChartManager.render('some-canvas', ...)` even if the canvas was just recreated milliseconds ago.
- localStorage keys: `cairo_atlas_layout_v1` for layout state. Bump the version (`_v2`, `_v3`) if you make a schema-breaking change.
- The drag handle is always a `<button>` element with `draggable="true"`. The slot itself is the drop target. Dragstart fires only on the handle (so accidental drags from anywhere else in the slot don't trigger reorders).

---

## 2026-05-06 — Initial git baseline + repair sprint

### Added
- Git repository initialized at the project root, pushed to `https://github.com/eliasfelix1000-web/Cairo-Atlas.git` (`main`).
- `.gitignore` covering local backups (`*.bak`), OS noise, IDE folders, node_modules, scratch logs.
- `README.md` — public-facing pitch, what the dashboard is, MCP requirements, how to run.
- `STACK.md` — runtime dependencies, MCP data sources, file map, entry-point line numbers.
- `ARCHITECTURE.md` — module/class layout, data-flow diagram, refresh-loop walkthrough, error-handling philosophy, known sharp edges.
- `CLAUDE.md` — onboarding for the next Claude session. Folds in the CĀIRO collaboration baseline (G's role, Claude's four hats, the working contract) adapted from Cowork to Claude Code primitives, plus a self-handoff "what I learned" section.
- `CHANGELOG.md` — this file.
- **Global error handlers** (`window.onerror`, `window.onunhandledrejection`) at the top of `<script>`. Anything that escapes a try/catch now lands in the `#error-log` panel via `atlasLog` instead of dying silently in devtools. *G's stated priority.*
- **`safeRender(name, fn)`** — wraps each renderer call inside `refresh()` so one renderer's failure logs with stack-trace head and is skipped; the rest of the pass continues.
- **`$(id)` and `on(id, ev, handler)` safe-DOM helpers** — every `getElementById` and `addEventListener` either succeeds or warns to the log panel. No more silent script aborts from one missing element.
- **`window.CairoAtlas` OOP facade** — class wrappers (`AtlasApi`, `Dashboard`, `ChatBubble`, `DateRangePicker`) over the existing `ATLAS`/`STATE`/`CHAT`/`DR` singletons. Pure delegation, no behavior change. The win is a single navigable surface for devtools and a clear handle for future sessions.
- **Top-of-`<script>` ToC comment** mapping all 12 logical sections in line order.

### Fixed
- **Dashboard not loading** *(SEV-1)* — `renderGsc()` (lines ~3346–3445 in the previous `index.html`) was missing both `wrap.innerHTML = html` and two closing braces, producing `SyntaxError: Unexpected end of input` at the end of the `<script>`. With the script failing to evaluate, none of the boot wiring ran and the dashboard appeared frozen on the loading skeletons. The block was already commented as legacy/removed and references an undefined `GSC` state object — it would still throw at runtime if the braces were closed. Excised the entire 247-line dead block (`_legacyGscRemoved_unused`, `readGscFile`, `parseGscText`, `GSC_IFRAME_SRC`, `renderGscIframe`, broken `renderGsc`).

### Changed
- **`refresh()`** — every renderer call now wrapped in `safeRender('name', fn)`. One render failure no longer aborts the whole pass.
- **Init IIFE hardened** — each step (localStorage prefs restore, topbar paint, chat restore, error-log paint, first refresh) independently guarded. Single failure can't dead-end boot.
- **Date picker, top controls, sort tables, chat wirings** — all bare `document.getElementById(…).addEventListener(…)` calls (~30 of them) replaced with `on(id, ev, handler)`. Missing element = warn, not throw.
- **Rel-time tick** — guarded the `#fresh-text` lookup so a missing element doesn't throw on every 5-second cycle.

### Removed
- The dead `renderGsc` block plus `_legacyGscRemoved_unused`, `readGscFile`, `parseGscText`, `GSC_IFRAME_SRC`, `renderGscIframe` (~247 lines). The Creative Analysis section (`CA`) replaced this functionality.

### Notes for next session
- `window.cowork.callMcpTool` and `window.cowork.askClaude` are the artifact-runtime globals; if Cowork renames them, every fetch + the chat dies. Worth adding a one-time runtime probe at boot.
- MCP tool names are hardcoded UUIDs (`mcp__3fe49b0d-…`) and may not match the registered names in the Cowork environment. If every Shopify call errors in the Logs panel, this is the most likely cause.
- Renderer duplication (`renderChannelTable` / `renderSourceTable` / `renderMediumTable` / `renderCampaignsTable`) is ripe for a `renderBreakdownTable` helper — would shrink the file by ~120 lines but is a separate commit, behind a manual visual-diff.
- See "What's still risky / left undone" in `CLAUDE.md` for the full list.
