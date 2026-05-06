# Changelog

Every patch on `main`, newest first. Each entry should answer "what + why" — file paths and line numbers belong in the diff, not here.

Format inspired by [Keep a Changelog](https://keepachangelog.com/) but loosened: pre-1.0, no SemVer yet.

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
