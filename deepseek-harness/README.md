# DeepSeek Harness（dsh）源码分析

> 本目录基于 `master` 分支快照（commit `b150a551b8`，tag `dsh-v0.1.1-rc.2`，版本 0.1.1-rc.2）整理，快照日期 2026-08-22。官方文档（docs/architecture.md 等）用于指路，**所有结论均以源码为准**。

## 阅读路径

| 文档 | 内容 |
|---|---|
| [01-core-functionality.md](./01-core-functionality.md) | 定位、设计哲学、核心功能、Agent=Model+Harness 拆解 |
| [02-architecture.md](./02-architecture.md) | Cordis 插件树、profile/bundle 分层、关键包 |
| [03-runtime.md](./03-runtime.md) | turn/step 循环、inbox 队列、工具调度管道、事件契约 |
| [04-session.md](./04-session.md) | SessionEvent 日志、派生上下文、fork/恢复、压缩 |
| [05-extensibility.md](./05-extensibility.md) | 一切皆插件：注册/作用域/事件/运行模式 |
| [06-ai-provider.md](./06-ai-provider.md) | LLM 适配层、认证、模型目录、流协议 |
| [07-security-eval.md](./07-security-eval.md) | 沙箱/审批/凭证/供应链 + 测试体系 |

## 核心结论（TL;DR）

- dsh（DeepSeek Harness）是 **"一切皆插件"** 的 agent harness：连模型适配、工具注册、会话日志、agent 循环本身都是 Cordis 插件，**没有特权核心**；扩展方式是把插件挂到配置树里，卸载时注册效果自动回滚（`docs/architecture.md` + `packages/boot/app-boot/src/profile.ts`）。
- 运行时由 **profile → bundle → cordis.patch.yml** 分层组装：`dsh-base` 是所有 profile 的第一层（模型/工具/会话/沙箱/审批/凭证/遥测），`dsh-web-app` 加浏览器 UI，`dsh-headless` 加一次性任务执行（`packages/bundle/base/cordis.patch.yml`、`packages/boot/app-boot/src/profile.ts:114`）。
- Agent 循环：**turn（≥0 个 step）/ step（一次模型请求 + 其工具调用）** 双层；输入走单一 inbox（`next-turn`/`next-step` 边界），`agent/pre-step`、`agent/request`、`agent/request-error`、`agent/turn-stopping` 是四个瀑布/串行扩展点（`packages/core/agent-loop/src/agent.ts:64`）。
- **会话日志是唯一事实源**：`SessionEvent` 追加式日志 + `deriveMessages()` 派生模型上下文；"模型可见即已落盘"是运行时不变式（`packages/core/session/src/types.ts:56`、`docs/architecture.md`）。本快照格式版本固定为 0，无迁移。
- 工具管道：`prepare（pre-execute 瀑布 + guard）→ dispatch（并行池/独占屏障，上限 10）→ finalize（post-execute）/ finish`，结果按模型顺序提交（`packages/core/agent-loop/src/tool-calls.ts:59`、`packages/core/tools/src/index.ts:451`）。
- 安全内置：默认 `workspace-write` 沙箱 + `ask` 审批（`packages/bundle/base/cordis.patch.yml`），子代理是原生能力（`ctx.subagents` 多 provider，含 ACP/Claude Code/Codex 桥）。
- 运行形态：`dsh web`（HTTP + 浏览器 UI）、`dsh --profile headless "task"`（无服务器一次性）、`dsh --dump-config`（打印配置树）、`dsh plugin`。

## 文档约定

- 代码路径相对 `../deepseek-harness`；关键结论附 `path:line`。
- 官方 `docs/architecture.md` 是很好的索引，但本文所有"已实现"结论均核对过源码；版本为 developer preview，破坏性变更频繁。
