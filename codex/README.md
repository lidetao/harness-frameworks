# Codex CLI 源码分析

> 本目录基于 `main` 分支快照（commit `4f39251a01`，`rusty-v8-v150.4.0-1014-g4f39251a01`）整理，快照日期 2026-08-22。仓库是 openai/codex（Rust 为主的 monorepo），**结论以源码为准**，README 宣传仅作背景。

## 阅读路径

| 文档 | 内容 |
|---|---|
| [01-core-functionality.md](./01-core-functionality.md) | 定位、哲学、核心功能、Agent=Model+Harness 拆解 |
| [02-architecture.md](./02-architecture.md) | 112 个 Rust crate 的分层、关键模块、依赖方向 |
| [03-runtime.md](./03-runtime.md) | turn 循环、采样、工具执行、hook 事件、审批/沙箱 |
| [04-session.md](./04-session.md) | rollout JSONL + SQLite、恢复/回滚、压缩、上下文 |
| [05-extensibility.md](./05-extensibility.md) | hooks、插件/marketplace、MCP、skills、子代理/角色 |
| [06-ai-provider.md](./06-ai-provider.md) | ModelProvider 抽象、认证、模型配置、SDK |
| [07-security-eval.md](./07-security-eval.md) | 权限模型、沙箱、供应链 + 测试体系 |

## 核心结论（TL;DR）

- Codex CLI 是 OpenAI 的**本地运行的编码 agent**（`codex-rs` Rust 工作区，112 个 crate；另有 TypeScript/Python SDK 与 npm 包装），主打 ChatGPT 账号登录 + OpenAI Responses API（含 WebSocket 实时）。
- 核心循环是**单会话多 step 的 turn 驱动**：`run_turn`（`codex-rs/core/src/session/turn.rs:153`）在"采样→工具→再采样"之间循环，支持运行中 steer（pending input）、mid-turn 自动压缩、turn-stop hooks。
- **权限是内置一等公民**：`AskForApproval`（Never/UnlessTrusted/OnRequest/Granular，`codex-rs/protocol/src/protocol.rs:924`）+ exec-policy 规则文件 + 沙箱（`sandbox_mode`，linux bwrap/Windows）——这是三框架中权限模型最重的。
- **持久化是 rollout JSONL + SQLite 双轨**：`~/.codex/sessions/<thread>/rollout-*.jsonl` 记录全部事件，`state_db`（SQLite）做索引/搜索/恢复（`codex-rs/rollout/src/recorder.rs`、`state_db.rs`）。
- 扩展面是 **hooks（SessionStart/UserPromptSubmit/PreToolUse/PostToolUse/Stop/SessionEnd/PermissionRequest/Compact）+ 插件/marketplace + MCP + skills**；子代理是原生能力（AgentRegistry/角色/builtin 模板）。
- 运行形态多：TUI（`codex-rs/tui`）、`exec`（非交互 JSONL）、review、MCP server、app-server/desktop、remote-control、exec-server、sandbox。

## 文档约定

- 代码路径相对 `../codex`（`codex-rs/...` 即 Rust 工作区）；关键结论附 `path:line`。
- 本快照为开发主线，与 npm 发布的稳定版可能不同。
