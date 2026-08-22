# 1. 核心功能与设计哲学

## 1.1 项目是什么

dsh（`@deepseek-ai/dsh`，DeepSeek Harness）是 DeepSeek AI 开发的**开源 agent harness**（MIT），当前处于 developer preview（0.1.1-rc.2）。它构建在 [Cordis](https://github.com/cordiverse/cordis) 之上——一个"插件贡献服务、类型化事件、可回滚效果到共享上下文"的运行时，官方论文视角是《A Programming Paradigm for Spatiotemporal Composability》。

与 pi（最小核心 + TypeScript 扩展）和 codex（单一厂商产品化 CLI）不同，dsh 的定位是**组合平台**：产品本身由插件组成，用户通过配置树（`cordis.patch.yml`）替换任何一行，包括模型适配器、工具、会话日志、循环本身。

## 1.2 设计哲学：一切皆插件，没有特权核心

`docs/architecture.md` 原文："There is no privileged core to patch: you extend dsh by mounting a plugin beside the others, and registrations are effects that unwind when their plugin unloads."源码佐证：

- `packages/core/agent-loop/src/index.ts:1-15`：循环本身是"concrete agent-loop plugin"，通过 `Service` 注册到 Cordis `Context`；
- `packages/core/agent/src/index.ts:34-44`：`ctx.agents`、`ctx.agent` 通过模块增强挂到 Cordis `Context`；
- 插件树的组装顺序见 [02-architecture.md](./02-architecture.md)。

含义：

1. **替换即扩展**：想换模型提供商→注册新 `ctx.llm` adapter；想换 shell→注册 `ctx.shell` backend；想加权限策略→监听 `tools/*` 或注册 guard；想改循环→替换 agent-loop 插件。
2. **作用域**：`dsh-scope` 提供按 agent 隔离的注册（`agent.ctx` 内注册只对该 agent 生效，`packages/core/scope/src/index.ts`）。
3. **可回滚效果**：插件卸载时其注册（服务、事件、工具、设置）自动 unwind，配置热重载（HMR 行在 base bundle 中默认挂载，web/headless 显式关闭）。

## 1.3 核心能力清单

| 能力 | 实现 | 说明 |
|---|---|---|
| 会话日志 | `packages/core/session/` | 追加式 `SessionEvent`，~50 种事件（见 04），格式版本 0 |
| Agent 循环 | `packages/core/agent-loop/src/agent.ts` | turn/step 驱动 + inbox + 4 个瀑布钩子 |
| 工具注册/调度 | `packages/core/tools/` | schema（JSON Schema/TS/Python 三语生成）+ 调度器 + guard |
| 系统提示组装 | `packages/core/system-prompt/` | 有序 section/context/tool schema/变量 |
| LLM 适配层 | `packages/llm/llm/` | `LlmAdapter` 注册表、流、重试、token 计量 |
| 默认工具集 | `packages/bundle/base/cordis.patch.yml` | bash/pwsh/jobs/fs/fs-search/skill/subagent 等 |
| 沙箱/审批 | `packages/sandbox/*`、`packages/approval` | read-only/workspace-write/danger-full-access + ask |
| 子代理 | `packages/subagent/*` | 多 provider：in-process/spawn/fork/ACP/Claude Code/Codex/dsh-sdk |
| MCP | `packages/mcp/mcp-client` | 内置 MCP 客户端 |
| 记忆/上下文 | `packages/context/*`、`packages/compaction/*` | 时间/指令/文件引用/会话引用上下文；compaction |
| 目标/计划/任务 | `packages/goal`、`packages/plan`、`packages/jobs`、`packages/schedule` | goal 驱动、plan mode、后台 jobs |
| UI | `apps/web` + `packages/client/ui-*` | 浏览器客户端（会话/工具/审批/设置/子代理等组件） |
| Code Mode | `packages/code-runtime/*`、`packages/core/tools/src/code-mode.ts` | 原生 vs code（JS 沙箱）工具呈现模式 |

## 1.4 Agent = Model + Harness 拆解

按本系列文档的术语，dsh 的拆解是：

- **Harness 提供**：会话日志（durable facts）、turn/step 驱动、inbox（steer/follow-up/inject）、工具注册与调度、system-prompt 组装、沙箱/审批、凭证、子代理、MCP、UI/API 网关、telemetry。
- **模型只负责**：在 `llm/stream` 上消费组装好的消息 + 工具 schema，产出 `assistant/chunk`（流）与 `tool-call` 块；失败以 `LlmFailure` 结构化返回。
- **不变式**："Model-visible means logged"——任何进入模型请求的内容必须能从 SessionEvent 日志重建（`docs/architecture.md`，`runtime-context.ts` 在循环内断言）。
