# 1. 核心功能与设计哲学

## 1.1 项目是什么

Codex CLI 是 OpenAI 的编码 agent，本地运行、默认连 OpenAI 后端（Responses API / ChatGPT 订阅 / WebSocket 实时），Apache-2.0。它首先是**产品化的 CLI**（TUI/非交互/云任务），同时把 harness 能力（循环、工具、权限、会话、hooks）全部开源在 `codex-rs`。

与 pi（可插拔最小核心）和 dsh（一切皆插件）相比，codex 是**单厂商纵深产品**：深度绑定 OpenAI 模型与账号体系，权限/沙箱/审批/guardian 等"安全面"是其最重的工程投入。

## 1.2 核心能力清单

| 能力 | 实现 | 说明 |
|---|---|---|
| TUI | `codex-rs/tui` | 交互式终端 UI（`run_main`，`tui/src/lib.rs:929`），多 app 视图 |
| 非交互 | `codex-rs/exec` | `codex exec`，`--json` 输出 JSONL 事件（`exec/src/lib.rs:3`） |
| turn 循环 | `core/src/session/turn.rs` | 采样↔工具循环 + steer + mid-turn 压缩 |
| 工具系统 | `core/src/tools/` | 注册表、并行执行、审批、沙箱、speculative/plan |
| 权限 | `core/src/exec_policy.rs` + `protocol.rs` | AskForApproval 四级 + exec-policy 规则 |
| 会话 | `codex-rs/rollout` + `core/src/session/` | rollout JSONL + SQLite state db |
| 压缩 | `core/src/compact.rs` | 自动/手动/远端压缩 |
| hooks | `codex-rs/hooks` + `core/src/hook_runtime.rs` | 8 类 hook 事件 |
| MCP | `core/src/session/mcp.rs` + `codex-rs/mcp-server` | MCP 客户端/服务端 |
| 插件 | `codex-rs/core-plugins` + `plugin` | marketplace/本地插件 |
| skills | `codex-rs/skills` | 技能加载/提示注入 |
| 记忆 | `codex-rs/memories` + `core/src/memories.rs` | 跨会话记忆（summarize_memories，client.rs:717） |
| 子代理 | `core/src/agent/` | AgentRegistry + 角色 + 控制（spawn/审批/residency） |
| 云任务 | `codex-rs/cloud-tasks`、`codex-cli/src/.../remote_control` | Codex Cloud 任务浏览/应用 |
| SDK | `sdk/typescript`、`sdk/python` | 嵌入 harness 的客户端 |

## 1.3 设计哲学

- **安全纵深优先**：命令先过 exec-policy（规则文件 + 启发式危险命令表，`exec_policy.rs:120` 起的 `DANGEROUS_COMMANDS` 类列表），再决定 Forbidden/Prompt/Allow；审批、沙箱、guardian（自动审核审批请求）层层叠加。
- **产品形态多而核心一**：TUI/exec/review/MCP-server/app-server 全部复用 `core` 的 `Session`/`run_turn`，没有模式间行为漂移（`cli/src/main.rs` Subcommand 列表）。
- **单厂商绑定下的可替代性**：`ModelProvider` trait 允许自定义 provider（lmstudio/ollama/bedrock/azure 内置），但默认路径深度绑定 OpenAI Responses API 与 ChatGPT 认证。

## 1.4 Agent = Model + Harness 拆解

- **Harness 提供**：turn 循环、step 上下文（world state/turn diff）、工具注册与并行调度、审批与沙箱、rollout 持久化、压缩、hooks、MCP、skills/插件、子代理、记忆、遥测。
- **模型负责**：在 Responses API 流上产出 response items（text/tool call/reasoning）；harness 把 `ResponseItem` 流翻译成会话事件（`try_run_sampling_request`，`turn.rs:2179`）。
- **上下文**：每次采样前 `clone_history().for_prompt(...)` 从会话历史构造请求（`turn.rs:355-365`），并记录 reference context item 支持增量注入。
