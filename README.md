<div align="center">

# Panel Kit

**The shared toolkit behind these panels.**

Panel Kit is the small bridge, theme, and build setup these plugins use to ship fast, dark-first sidebar panels under 64 KiB (65,536 B). Use it if you're building your own Antigravity panel and want it to feel native.

[Homepage](https://eatingdream.dev) · English | [简体中文](README.zh-CN.md) · [Releases](https://github.com/eatingDream-antigravity-plugins/panel-kit/releases) · [AI Guide](docs/ai-guide.md) · [`llms.txt`](llms.txt)

[![CI](https://img.shields.io/github/actions/workflow/status/eatingDream-antigravity-plugins/panel-kit/ci.yml?branch=main&style=flat-square&label=CI&color=12151E&labelColor=181C28)](https://github.com/eatingDream-antigravity-plugins/panel-kit/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/eatingDream-antigravity-plugins/panel-kit?style=flat-square&color=12151E&labelColor=181C28)](https://github.com/eatingDream-antigravity-plugins/panel-kit/releases)
![Rust 1.75+](https://img.shields.io/badge/Rust-1.75%2B-12151E?style=flat-square&logo=rust&logoColor=F2F4F8)
[![License: MIT](https://img.shields.io/badge/license-MIT-2952CC?style=flat-square)](LICENSE)

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/architecture-dark.svg" />
  <source media="(prefers-color-scheme: light)" srcset="docs/architecture-light.svg" />
  <img src="docs/architecture-dark.svg" alt="Panel Kit architecture: Rust MCP runtime and Lucid Indigo panel UI modules synced via SHA-256 manifest into Context Radar, Clam Guard, Remote Cast, and Colab Deck" width="760" />
</picture>

</div>

<sub>Part of the <a href="https://github.com/eatingDream-antigravity-plugins">Antigravity Plugins</a> suite: <a href="https://github.com/eatingDream-antigravity-plugins/context-radar">Context Radar</a> · <a href="https://github.com/eatingDream-antigravity-plugins/clam-guard">Clam Guard</a> · <a href="https://github.com/eatingDream-antigravity-plugins/remote-cast">Remote Cast</a> · <a href="https://github.com/eatingDream-antigravity-plugins/colab-deck">Colab Deck</a> · <strong>Panel Kit</strong></sub>

## Built on Panel Kit

| Plugin | What it solves |
| :--- | :--- |
| [**Context Radar**](https://github.com/eatingDream-antigravity-plugins/context-radar) | Know how full your context is before it bites — live token, cache, and cost telemetry with guided handoffs. |
| [**Clam Guard**](https://github.com/eatingDream-antigravity-plugins/clam-guard) | Run agents with the lid closed, without cooking your Mac — thermal guard, safe brightness, and verified screen lock. |
| [**Remote Cast**](https://github.com/eatingDream-antigravity-plugins/remote-cast) | Your Chrome, live inside Antigravity — stream and steer everyday or isolated Chrome tabs right in the sidebar. |
| [**Colab Deck**](https://github.com/eatingDream-antigravity-plugins/colab-deck) | Colab GPUs from your sidebar, with a spending limit — manage sessions, port forwarding, and auto-stop rules. |

## Is It for You?

- **Use Panel Kit** if you ship local Rust MCP servers with embedded HTML panels and want `< 1 ms` warm reads, zero white-flash startup, a shared Lucid Indigo theme, and automated CSP, size, and EN/ZH copy checks across multiple repos.
- **Skip it** if you only need a one-off script or a browser-hosted web app with a full bundler and runtime dependencies.
- **Why file sync instead of a git crate?** Each plugin stays 100% standalone for offline `cargo build --release --locked` with zero shared lockfile coupling, while SHA-256 drift checks catch local edits in CI.

## What You Get

| | |
| :--- | :--- |
| **MCP Protocol (`src/mcp_protocol.rs`)** | Protocol version negotiation, strict UI capability checks, shared `tools/list` + `tools/call` visibility rules, and JSON-RPC 2.0 single/batch dispatch. |
| **Warm SWR Cache (`src/swr_cache.rs`)** | `< 1 ms` cached reads with deduplicated background refreshes, bounded cold-start waits, and revision-based unchanged checks. |
| **Safe HTML Boot (`src/html_boot.rs`)** | File `(mtime, size)` template cache with `</script>` and `<!--` safe JSON escaping for embedded single-file panels. |
| **Safe Test Guards (`src/test_guard.rs`)** | Compile-time test-mode blockers preventing unit tests from mutating displays, browsers, or live host state. |
| **Lucid Indigo Theme & Bridge (`panel/tokens.css`, `panel/bridge.ts`)** | Dark/light Lucid Indigo tokens (`#0B0D12` / `#F8FAFD`, Cobalt `#6895FF` / `#2952CC`, `--bd` backdrop), strict parent PostMessage origin checks, 3-mode host theme sync (`system` / `light` / `dark`), and net RTT timing. |
| **Browserless QA & i18n Gate (`panel/guard.ts`)** | Enforces `<= 64 KiB` (`65,536 B`, `56 KiB` warn) inline `<script>` / `<script type="module">` CSP bounds, Dark & Light Byte-0 SSR root locks, Lucid Indigo token checks, plus `checkI18nParity` and `checkEnNoCjk` for EN/ZH copy. |

## Quick Start

**1 · Verify the kit locally**

```sh
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test --locked
npm ci --ignore-scripts && npm run typecheck && npm test
./scripts/sync-kit.sh --check-manifest && ./scripts/check_sanitized.sh
```

**2 · Sync the unified Lucid Indigo panel kit into one or all plugins and verify zero drift**

```sh
./scripts/sync-kit.sh --panel ../context-radar ../clam-guard ../remote-cast ../colab-deck
./scripts/sync-kit.sh --check ../context-radar ../clam-guard ../remote-cast ../colab-deck
```

**3 · Validate a built single-file panel HTML and EN/ZH copy**

```sh
node panel/guard.ts ../clam-guard/panel/dist/panel.html --max-bytes 65536
node -e 'import("./panel/guard.ts").then(g => { g.assertI18nParity(en, zh); g.assertEnNoCjk(en); })'
```

<sub>Every sync writes SHA-256 digests to <code>.agy-kit-sync.json</code> so CI (<code>./scripts/sync-kit.sh --check &lt;repo&gt;</code>) catches drift while keeping each plugin's <code>Cargo.lock</code> standalone. Use <code>--files ...</code> or <code>--map src=dest</code> for custom subsets, and run <code>./scripts/sync-kit.sh --update-manifest</code> after editing canonical kit files.</sub>

## For AI Agents

All Rust signatures, JS bridge options, `checkI18nParity` / `checkEnNoCjk` APIs, and `--panel` / `--files` / `--map` sync flags live in the agent reference:

> *"Read [`llms.txt`](llms.txt) and [`docs/ai-guide.md`](docs/ai-guide.md) before syncing modules or wiring `agy-mcp-kit` checks into an Antigravity plugin."*

## Safety & Privacy

- **Zero runtime network calls**: Neither the Rust crate nor the JS panel kit makes outbound network requests.
- **Strict PostMessage checks**: `panel/bridge.ts` drops events where `ev.source !== window.parent` and locks `targetOrigin` after initialization.
- **Redacted secret & path scan**: `scripts/check_sanitized.sh` scans tracked files and the staged index, printing `file:line` only without echoing matched text.
- **Zero-side-effect tests**: `test_guard.rs` blocks live OS, display, browser, or billed cloud effects during tests (`AGY_TEST_GUARD_DENY=1` for subprocesses).

## Documentation

- [AI Guide & API Reference (`docs/ai-guide.md`)](docs/ai-guide.md) · [LLM Index (`llms.txt`)](llms.txt) · [Kit Manifest (`kit-manifest.json`)](kit-manifest.json)
- [Contributing](CONTRIBUTING.md) · [Security](SECURITY.md)

## License

[MIT](LICENSE)
