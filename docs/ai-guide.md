# Panel Kit (`agy-mcp-kit`) — AI Agent Guide & API Reference

**Panel Kit** ([`eatingDream-antigravity-plugins/panel-kit`](https://github.com/eatingDream-antigravity-plugins/panel-kit), crate `agy-mcp-kit`, <https://eatingdream.dev>) uses a **canonical source + standalone vendored sync** workflow:
- **Canonical Source**: All runtime primitives and Lucid Indigo panel standards live and are tested here (`cargo test --locked`, `npm run typecheck && npm test` running `node --test panel/test/kit.test.ts`).
- **Standalone Sync**: Downstream plugins — [Context Radar](https://github.com/eatingDream-antigravity-plugins/context-radar), [Clam Guard](https://github.com/eatingDream-antigravity-plugins/clam-guard), [Remote Cast](https://github.com/eatingDream-antigravity-plugins/remote-cast), and [Colab Deck](https://github.com/eatingDream-antigravity-plugins/colab-deck) — vendor only the files they need via `scripts/sync-kit.sh`, verified by SHA-256 digests in `.agy-kit-sync.json` and `kit-manifest.json`.

---

## 1. Rust Runtime Modules (`src/`)

Only depends on `serde` and `serde_json`. Every module is self-contained so it compiles cleanly whether imported from the `agy_mcp_kit` crate or vendored under `src/kit/` in a downstream plugin.

### `src/mcp_protocol.rs`
- **Constants**:
  - `SUPPORTED_PROTOCOL_VERSIONS = &["2025-11-25", "2025-06-18", "2025-03-26", "2024-11-05"]`
  - `LATEST_PROTOCOL_VERSION = "2025-11-25"`
  - `UI_EXTENSION = "io.modelcontextprotocol/ui"`
  - `APP_MIME = "text/html;profile=mcp-app"`
  - Error codes: `PARSE_ERROR (-32700)`, `INVALID_REQUEST (-32600)`, `METHOD_NOT_FOUND (-32601)`, `INVALID_PARAMS (-32602)`, `INTERNAL_ERROR (-32603)`
- **Session & Negotiation**:
  - `McpSession` (`Copy`, `Default`):
    - `session.try_negotiate(&init_params) -> Result<(&'static str, bool), &'static str>` — strict MCP `initialize` validation (`protocolVersion`, `capabilities`, `clientInfo.name`); rejects duplicate `initialize` once `session.initialized == true`.
    - `session.negotiate(&init_params) -> (&'static str, bool)` — permissive helper (preserves frozen negotiated state once `session.initialized == true`).
    - `session.supports_jsonrpc_batch() -> bool` — returns `true` only on legacy `2025-03-26` / `2024-11-05` sessions.
  - `negotiate_protocol_version(Option<&str>) -> &'static str`
  - `client_supports_mcp_apps(&init_params) -> bool` — strict SEP-1865 check requiring `capabilities.extensions["io.modelcontextprotocol/ui"].mimeTypes` to be an array explicitly containing `"text/html;profile=mcp-app"`.
  - `build_initialize_result(name, version, protocol_version, apps_capable, extra_capabilities) -> Value` — merges sibling `extensions` cleanly and strips `UI_EXTENSION` whenever `apps_capable == false`.
- **Shared Tool Visibility (`tools/list` & `tools/call`)**:
  - `ToolCaller::{Model, App}`
  - `ToolVisibility::{ModelAndApp, ModelOnly, AppOnly, Hidden}` + `ToolVisibility::from_tool_json(&tool)` — defaults to `ModelAndApp` only when `_meta.ui.visibility` is absent; fails closed to `Hidden` whenever `visibility` is present with a non-array or contains neither `"model"` nor `"app"`.
  - `filter_tools_for_session(tools: &[Value], apps_capable: bool) -> Vec<Value>`
  - `is_tool_allowed_in_catalog(tool_name: &str, tools: &[Value], apps_capable: bool) -> bool`
  - `is_tool_callable_by_in_catalog(tool_name: &str, tools: &[Value], caller: ToolCaller, apps_capable: bool) -> bool`
- **JSON-RPC 2.0 & MCP Dispatch**:
  - `dispatch_mcp_jsonrpc(session: McpSession, msg: &Value, handler) -> Option<Value>` — takes `McpSession` by value (`Copy`, avoiding borrow conflicts when `handler` mutates `session`), enforces string/integer MCP IDs, rejects pre-initialization requests (except `initialize`/`ping`), rejects duplicate `initialize`, forbids batched `initialize`, and rejects JSON-RPC batches on MCP `>= 2025-06-18`.
  - `dispatch_jsonrpc(msg: &Value, handler) -> Option<Value>` — generic JSON-RPC 2.0 dispatcher for single requests, notifications (returns `None`), and batch arrays `[...]` (empty `[]` returns `-32600`; notification-only batch returns `None`).
  - `dispatch_jsonrpc_str(raw: &str, handler) -> Option<Value>` — returns `-32700` on invalid JSON.

### `src/swr_cache.rs`
- `SwrConfig { fresh_ttl, cold_wait_timeout, error_backoff, max_stale_ttl, thread_name }`
- `SwrSnapshot<T> { value: Arc<T>, rev: u64, stale: bool }`
- `SwrCache::new(config, loader)`
  - `prewarm()` — spawns at most 1 background refresh thread without blocking.
  - `get_or_trigger_refresh() -> Result<SwrSnapshot<T>, String>` — returns `<1 ms` on warm hits (triggering single-flight background refresh if stale), enforces optional `max_stale_ttl` bounded staleness, coalesces concurrent cold callers with `checked_add`-safe `cold_wait_timeout`, and re-spawns a fresh generation refresh if `invalidate()` runs during a cold wait.
  - `force_refresh_async() -> bool` — triggers background refresh even while fresh.
  - `invalidate()` — marks stale and bumps generation (fencing stale in-flight completions and panics) without dropping the cached value.
  - `set_value(value: T) -> u64` — installs a fresh value immediately (e.g. after mutations), increments `rev`, fences older in-flight refreshes, and drops replaced values outside the mutex.
  - `current_rev() -> u64`, `is_unchanged(since_rev) -> bool`, `is_unchanged_and_revalidate(since_rev) -> bool` — enables 0-byte unchanged polling.

### `src/html_boot.rs`
- `CompiledTemplate::from_html(html, placeholder)`
  - `render_with_boot(boot_script) -> String`
  - `try_render_with_global_json(global_var, &data) -> Result<String, String>` — rejects empty or prototype-polluting keys (`__proto__`, `constructor`, `prototype`) and falls back to JSON-escaped `window["..."]` for non-identifier keys.
  - `render_with_global_json(global_var, &data) -> String` — infallible wrapper falling back to `/* invalid boot global */` on rejected keys.
- `is_safe_js_identifier(name: &str) -> bool` — validates plain ASCII JS identifiers while rejecting prototype-polluting keys and ES reserved keywords.
- `HtmlLoader::new(embedded_html, placeholder, candidate_paths)` & `HtmlLoader::with_env_and_exe_fallback(...)`
  - `prewarm()`, `load() -> Arc<CompiledTemplate>`, `render_boot(global_var, &data) -> String`
  - Opens each candidate `File` once (`file.metadata()` + `4 MiB` cap) and caches by `(path, mtime_ns, size)` with zero disk reads on warm hits.
- `escape_json_for_inline_script(raw_json)` & `serialize_boot_json(&val)` — escapes `<`, `>`, `&`, `U+2028`, `U+2029` to neutralize `</script>` and `<!--`.

### `src/test_guard.rs`
- `ensure_not_in_test(action: &str) -> Result<(), String>` — fails at compile time under `#[cfg(test)]` (when vendored in the target crate) and at runtime when `AGY_TEST_GUARD_DENY=1`.
- `guard_host_effect!("action")` — macro expanding `#[cfg(test)]` at the caller's crate site when consumed as a library dependency.
- **Integration test note (`tests/*.rs`)**: Cargo compiles library crates without `#[cfg(test)]` when building `tests/*.rs` integration tests, so set `AGY_TEST_GUARD_DENY=1` in integration tests (or use `guard_host_effect!`) to block compiled-lib host side effects.

---

## 2. Canonical Panel Kit (`panel/`)

### `panel/tokens.css`
Single Source of Truth stylesheet defining all 22 **Lucid Indigo** two-letter tokens (`--bg`, `--s1`, `--s2`, `--s3`, `--s0`, `--t1`, `--t2`, `--t3`, `--ln`, `--l2`, `--ls`, `--sh`, `--ac`, `--ah`, `--as`, `--ab`, `--cb`, `--ct`, `--ok`, `--wn`, `--er`, `--in`) plus `--tk`, `--sn`, `--sf`, `--mn`:
- Dark default `:root`: `#0B0D12` canvas, `#12151E` surface, `#181C28` well, `#1D2231` hover, `#F2F4F8` ink, Precision Cobalt `--ac: #6895FF` (`--ah: #82A8FF`), high-contrast CTA `--cb: #F2F4F8` / `--ct: #0B0D12`, status `--ok: #34D399`, `--wn: #FBBF24`, `--er: #F87171`.
- Light `:root[data-theme="light"]`: `#F8FAFD` canvas, `#FFFFFF` surface, `#EEF2F8` well, `#E4EAF4` hover, `#0F1420` ink, Swiss Cobalt `--ac: #2952CC` (`--ah: #1F42AA`), high-contrast CTA `--cb: #0F1420` / `--ct: #FFFFFF`, status `--ok: #059669`, `--wn: #B45309`, `--er: #DC2626`.
- Includes `html, body { background: var(--bg); color: var(--t1); }` (never `--color-background-primary`), universal Frame-0 boot shell classes (`.boot-frame`, `.boot-bar`, `.boot-dot`), `@media (prefers-reduced-motion: reduce)`, and `:lang(zh-CN)` label reset; zero `prefers-color-scheme`, zero `opacity: 0` keyframes.
- **How plugins consume it**: Sync via `./scripts/sync-kit.sh --panel <plugin-repo>` into `panel/kit/tokens.css`, then prepend `panel/kit/tokens.css` ahead of plugin-local `styles.css`/`theme.css` inside `pack.ts`/`build.ts` and remove duplicate `:root` token blocks from the plugin CSS.

### `panel/bridge.ts`
- `LATEST_PROTOCOL_VERSION = "2026-01-26"`
- `App(appInfo, appCapabilities, options)`:
  - Strict PostMessage check: drops any event where `!target || ev.source !== target` (`target = window.parent` when `window.parent !== window`).
  - `options.targetOrigin`: pins expected host origin (`"*"` by default; when `"*"`, automatically locks inbound/outbound origin to `ev.origin` after `ui/initialize` whenever `ev.origin !== "null"`).
  - `connect({ retries, retryDelayMs, timeoutMs })`: coalesced single-flight `ui/initialize` handshake validating exact `protocolVersion === "2026-01-26"`; retryable after failure or `close()`, with immediate timer cancellation on `close()`.
  - `request(method, params, { timeoutMs, maxTimeoutMs })`: rejects immediately before `connect()` or after `close()` (except `ui/initialize`), validates finite `timeoutMs`, and scopes `notifications/progress` timer resets to matching `progressToken`.
  - `hasServerTools()`: checks `hostCapabilities.serverTools`.
  - `callServerTool(params, opts)` & `callServerToolWithMetrics(params, opts)` -> `{ result, totalMs, serverMs, netRttMs }`, plus `getLastRttMs()` and `measureNetRttMs(totalMs, serverMs)`.
  - `onhostcontextchanged(delta, merged)`: invoked after deep-merging `hostCtx.styles.{variables,css}` and applying live theme/luminance (`applyHostContext`, `applyHostStyleVariables` with strict fail-closed `parsed.a >= 0.9 && parsed.lum !== null` guard on `--color-background-primary`; `applyDocumentTheme` setting literal `#0B0D12` / `#F8FAFD` and deferring `"light"` via `requestAnimationFrame` while `.boot-frame` is still mounted in `#app`).
  - `autoResize` defaults to `false` and only sends `ui/notifications/size-changed` when `autoResize: true` AND `hostContext.containerDimensions.maxHeight != null` (zero spam in side panes).

### `panel/guard.ts`
- ESM API: `assertPanelInvariants(html, { maxBytes = 65536, hardMaxBytes = 65536, warnBytes = 57344, bootPlaceholder = null, skipJsParse = false, allowEmptyAppRoot = false })`, `checkPanelInvariants(html, opts)`, `checkI18nParity(enMap, zhMap)`, `assertI18nParity(enMap, zhMap)`, `checkEnNoCjk(enMap, { allowKeys })`, and `assertEnNoCjk(enMap, { allowKeys })`.
- CLI: `node panel/guard.ts <path/to/panel.html> [--max-bytes 65536] [--allow-empty-app-root]` (fails closed with exit code `2` on unknown `--*` flags).
- Enforces: byte budget (`64 KiB` / `65,536` B hard, `56 KiB` / `57,344` B warn), single inline `<script>` (syntax-checked), HTML comment stripping before tag checks, strict directive-bounded `<head>` CSP `default-src 'none'`, 0 external URLs (`http(s)://`, `ws(s)://`, protocol-relative `//` in HTML/CSS/JS), dark-first `<html>`/`<meta>`/`<body>` `#0B0D12` locks, 0 `--color-background-primary` on `html`/`body` background, non-empty Frame-0 `.boot-frame` shell inside `<div id="app">` before `<script>` (unless `allowEmptyAppRoot: true`), 0 `prefers-color-scheme`, 0 `opacity:0`/`0%`/`!important` in `@keyframes`/`@-webkit-keyframes`, 0 `innerHTML`/`outerHTML` (`=`/`+=`/bracket) or `insertAdjacentHTML`/`document.write`/`createContextualFragment`/`parseFromString`, 0 `--buncss-*`, all 22 Lucid Indigo tokens in both themes across concatenated comment-stripped `<style>` blocks + canonical dark (`--bg: #0B0D12`, `--ac: #6895FF`, `--ok: #34D399`) and light (`--bg: #F8FAFD`, `--ac: #2952CC`, `--ok: #059669`) values, `prefers-reduced-motion: reduce`, 1:1 EN/ZH translation key & `{placeholder}` parity, and zero stray CJK characters in EN copy.

---

## 3. Sync & Drift Tooling (`scripts/sync-kit.sh`)

```sh
# Refresh kit-manifest.json after editing canonical kit files
./scripts/sync-kit.sh --update-manifest

# Verify kit-manifest.json matches disk (runs in CI)
./scripts/sync-kit.sh --check-manifest

# Sync unified Lucid Indigo panel kit (tokens.css, bridge.ts, guard.ts) into all 4 plugins at once
./scripts/sync-kit.sh --panel ../context-radar ../clam-guard ../remote-cast ../colab-deck

# Sync all canonical kit files into a target repo
./scripts/sync-kit.sh /path/to/plugin-repo

# Sync a selective subset with custom destination mapping
./scripts/sync-kit.sh \
  --files src/mcp_protocol.rs,panel/tokens.css,panel/bridge.ts,panel/guard.ts \
  --map panel/bridge.ts=panel/bridge.ts \
  /path/to/plugin-repo

# Re-sync using the target repo's existing .agy-kit-sync.json file mapping
./scripts/sync-kit.sh /path/to/plugin-repo

# Verify zero drift across one or more plugins in CI or pre-push
./scripts/sync-kit.sh --check ../context-radar ../clam-guard ../remote-cast ../colab-deck
```

Default destination map:
- `src/*.rs` -> `src/kit/*.rs`
- `panel/{tokens.css,bridge.ts,guard.ts}` -> `panel/kit/{tokens.css,bridge.ts,guard.ts}`
- `scripts/check_sanitized.sh` -> `scripts/check_sanitized.sh`
- `.github/sanitize-patterns.txt` -> `.github/sanitize-patterns.txt`
- `LICENSE` -> `THIRD_PARTY_LICENSES/agy-mcp-kit-LICENSE`
