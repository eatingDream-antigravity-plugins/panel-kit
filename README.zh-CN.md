<div align="center">

# Panel Kit

**这些侧边栏面板背后的共享套件。**

Panel Kit 是这组插件共用的桥接层、主题令牌与构建校验套件，用来将打包体积控制在 64 KiB（65,536 B）以内、深色优先、零白屏闪烁的侧边栏面板。如果你也在开发自己的 Antigravity 面板，想让它用起来像原生组件一样顺手，可以直接接入这套工具。

[主页](https://eatingdream.dev) · [English](README.md) | 简体中文 · [Releases](https://github.com/eatingDream-antigravity-plugins/panel-kit/releases) · [AI 指南](docs/ai-guide.md) · [`llms.txt`](llms.txt)

[![CI](https://img.shields.io/github/actions/workflow/status/eatingDream-antigravity-plugins/panel-kit/ci.yml?branch=main&style=flat-square&label=CI&color=12151E&labelColor=181C28)](https://github.com/eatingDream-antigravity-plugins/panel-kit/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/eatingDream-antigravity-plugins/panel-kit?style=flat-square&color=12151E&labelColor=181C28)](https://github.com/eatingDream-antigravity-plugins/panel-kit/releases)
![Rust 1.75+](https://img.shields.io/badge/Rust-1.75%2B-12151E?style=flat-square&logo=rust&logoColor=F2F4F8)
[![License: MIT](https://img.shields.io/badge/license-MIT-2952CC?style=flat-square)](LICENSE)

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/architecture-dark.svg" />
  <source media="(prefers-color-scheme: light)" srcset="docs/architecture-light.svg" />
  <img src="docs/architecture-dark.svg" alt="Panel Kit 架构图：Rust MCP 运行时与静夜靛蓝面板套件通过 SHA-256 清单零漂移同步至 Context Radar、Clam Guard、Remote Cast 与 Colab Deck" width="760" />
</picture>

</div>

<sub>属于 <a href="https://github.com/eatingDream-antigravity-plugins">Antigravity 插件套件</a>：<a href="https://github.com/eatingDream-antigravity-plugins/context-radar">Context Radar</a> · <a href="https://github.com/eatingDream-antigravity-plugins/clam-guard">Clam Guard</a> · <a href="https://github.com/eatingDream-antigravity-plugins/remote-cast">Remote Cast</a> · <a href="https://github.com/eatingDream-antigravity-plugins/colab-deck">Colab Deck</a> · <strong>Panel Kit</strong></sub>

## 基于 Panel Kit 构建的插件

| 插件 | 解决什么问题 |
| :--- | :--- |
| [**Context Radar**](https://github.com/eatingDream-antigravity-plugins/context-radar) | 在上下文撑爆前看清用量：实时查看 token、缓存命中与费用，并一键接续新会话。 |
| [**Clam Guard**](https://github.com/eatingDream-antigravity-plugins/clam-guard) | 合上 MacBook 机盖也能安心跑 Agent：防休眠、温控降载、安全调亮度与锁屏保护。 |
| [**Remote Cast**](https://github.com/eatingDream-antigravity-plugins/remote-cast) | 把 Chrome 实时搬进 Antigravity 侧边栏：观看并操控日常或独立浏览器标签页。 |
| [**Colab Deck**](https://github.com/eatingDream-antigravity-plugins/colab-deck) | 在侧边栏管 Colab GPU 并设好花费上限：启停会话、端口转发与自动停止保护。 |

## 是否适合你？

- **适合使用 Panel Kit**：你正在构建带内嵌 HTML 面板的本地 Rust MCP 服务，希望开箱获得 `< 1 ms` 热读取、零白屏启动、统一的静夜靛蓝主题，以及跨仓库自动化的 CSP、体积与中英文文案校验。
- **无需使用**：你只需要一次性脚本，或基于完整前端构建器与运行时依赖的纯网页应用。
- **为什么用文件同步而非 Git Crate？** 每个插件的离线 `cargo build --release --locked` 都保持 100% 独立、不互相锁死 `Cargo.lock`，同时 CI 通过 SHA-256 校验确保零漂移。

## 核心能力

| | |
| :--- | :--- |
| **MCP 协议（`src/mcp_protocol.rs`）** | 协议版本协商、严格 UI 扩展校验、`tools/list` 与 `tools/call` 共用可见性规则，以及 JSON-RPC 2.0 单条/批处理分发。 |
| **热读 SWR 缓存（`src/swr_cache.rs`）** | `< 1 ms` 缓存热读、后台去重刷新、有界冷启动等待与基于版本号的无变更快速返回。 |
| **安全 HTML 注入（`src/html_boot.rs`）** | 基于 `(mtime, size)` 的模板缓存，并对 `</script>` 与 `<!--` 做安全 JSON 转义。 |
| **零副作用测试护栏（`src/test_guard.rs`）** | 编译期测试模式拒绝器，防止单元测试意外改动屏幕、浏览器或真实宿主机状态。 |
| **静夜靛蓝主题与桥接（`panel/tokens.css`、`panel/bridge.ts`）** | 深浅色静夜靛蓝令牌（`#0B0D12` / `#F8FAFD`，钴蓝 `#6895FF` / `#2952CC`，`--bd` 遮罩）、严格父窗口 PostMessage 校验、三态宿主主题同步（`system` / `light` / `dark`）与净 RTT 测量。 |
| **无浏览器 QA 与 i18n 门禁（`panel/guard.ts`）** | 强制 `<= 64 KiB`（`65,536 B`，`56 KiB` 预警）内联 `<script>` / `<script type="module">` CSP 预算、深浅色首字节 SSR 锁定与令牌检查，并提供 `checkI18nParity` 与 `checkEnNoCjk` 中英文文案校验。 |

## 快速开始

**1 · 本地运行全部校验**

```sh
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test --locked
npm ci --ignore-scripts && npm run typecheck && npm test
./scripts/sync-kit.sh --check-manifest && ./scripts/check_sanitized.sh
```

**2 · 一键同步静夜靛蓝面板套件到目标插件仓库并校验零漂移**

```sh
./scripts/sync-kit.sh --panel ../context-radar ../clam-guard ../remote-cast ../colab-deck
./scripts/sync-kit.sh --check ../context-radar ../clam-guard ../remote-cast ../colab-deck
```

**3 · 校验编译后的单文件 `panel.html` 与中英文文案**

```sh
node panel/guard.ts ../clam-guard/panel/dist/panel.html --max-bytes 65536
node -e 'import("./panel/guard.ts").then(g => { g.assertI18nParity(en, zh); g.assertEnNoCjk(en); })'
```

<sub>每次同步都会将 SHA-256 摘要写入目标仓库的 <code>.agy-kit-sync.json</code>，CI 运行 <code>./scripts/sync-kit.sh --check &lt;repo&gt;</code> 即可捕获漂移，同时保持各插件 <code>Cargo.lock</code> 完全独立。支持 <code>--files ...</code> 或 <code>--map src=dest</code> 自定义同步范围，修改源套件文件后请运行 <code>./scripts/sync-kit.sh --update-manifest</code>。</sub>

## 给 AI Agent 使用

完整的 Rust 类型签名、JS 桥接参数、`checkI18nParity` / `checkEnNoCjk` API 与 `--panel` / `--files` / `--map` 同步细节见 Agent 参考文档：

> *「请先阅读 [`llms.txt`](llms.txt) 和 [`docs/ai-guide.md`](docs/ai-guide.md)，再将 `agy-mcp-kit` 模块与校验门禁同步接入目标插件。」*

## 安全与隐私边界

- **运行时零主动联网**：Rust 模块与 JS 面板套件均不发起任何出站网络请求。
- **严格 PostMessage 来源校验**：`panel/bridge.ts` 丢弃一切 `ev.source !== window.parent` 的消息，并在初始化后锁定 `targetOrigin`。
- **脱敏密钥与路径扫描**：`scripts/check_sanitized.sh` 同时扫描工作区与暂存区，仅输出 `file:line` 而不回显敏感行内容。
- **零副作用测试护栏**：`test_guard.rs` 在测试期间拦截真实桌面、显示、浏览器或计费云调用（子进程可设 `AGY_TEST_GUARD_DENY=1`）。

## 文档索引

- [AI Agent 指南与 API 参考（`docs/ai-guide.md`）](docs/ai-guide.md) · [LLM 索引（`llms.txt`）](llms.txt) · [套件清单（`kit-manifest.json`）](kit-manifest.json)
- [参与贡献](CONTRIBUTING.md) · [安全说明](SECURITY.md)

## 许可证

[MIT](LICENSE)
