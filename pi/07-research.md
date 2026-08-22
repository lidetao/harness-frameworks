# 7. 网上资料对照、生态与调研方向

> 本文件内容来自公开网络资料（2026-08 快照），用于交叉验证源码分析，标注"网上资料"的信息未在本仓库源码中逐一核对。

## 7.1 公开资料与源码的对照结论

| 公开资料说法 | 源码验证 |
|---|---|
| "四工具核心：read/write/edit/bash" | 一致：`defaultActiveToolNames = ["read","bash","edit","write"]`，另有可选只读工具 grep/find/ls |
| "系统提示词 < 1000 token" | 部分一致：`buildSystemPrompt` 刻意精简，工具列表按需注入（有 `toolSnippets` 逐行描述机制） |
| "三层架构：ai → agent → coding-agent" | 完全一致：依赖图单向，coding-agent 是唯一胶水层 |
| "会话是树形 JSONL，支持分支/fork" | 完全一致：`id/parentId` 树 + `buildSessionContext` 叶子回溯 |
| "压缩三触发：手动/阈值/溢出" | 完全一致：`compact()`/`_checkCompaction`/`_overflowRecoveryAttempted` |
| "扩展两阶段：Loader 注册、Runner 绑定" | 完全一致：`extensions/loader.ts` + `extensions/runner.ts`（`bindCore`） |
| "默认无 MCP/子代理/权限弹窗/plan mode" | 一致：这些能力通过扩展事件（`tool_call` 拦截等）实现；注意**项目信任门禁**是内置的（`trust-manager.ts`），与"逐工具权限弹窗"是两回事 |
| "15+ providers、订阅制 OAuth" | 一致：`providers/` 下 86 个 ts 文件（约 45 个工厂/注册文件）；Anthropic Pro/OpenAI Plus/Copilot 走 OAuth 订阅 |
| "AgentHarness 是测试/评估基座" | **不准确**：`AgentHarness` 是新一代 durable harness API（lane/session/records，多数方法 `HarnessNotImplemented`）；`pi-evals` 实际适配 coding-agent `AgentSession`（`packages/evals/src/pi-harness.ts`） |
| "会话只有一套 JSONL" | 不完全：产品层 `SessionManager` v3 之外，agent-core harness 还有 v4 JSONL / memory / SQLite 三后端 + records 日志（见 04-session.md 4.8） |
| "无任何权限防护" | 不完全：无逐工具权限 UI，但 bash 输出有清洗/截断（`bash-executor.ts`），项目资源加载有信任门禁（`project-trust.ts`），沙箱以扩展/容器提供（`examples/extensions/sandbox`、`docs/containerization.md`） |
| "pi 是 libGDX 作者创建、MIT、零后端" | 网上资料；package.json 作者即 Mario Zechner（badlogic） |

## 7.2 生态与周边项目（网上资料）

- **pi-chat**：Slack/聊天自动化封装，把消息转给编码 agent（官方 earendil-works 仓库）；
- **pi-share-hf**：把 PI 会话发布到 Hugging Face 数据集（badlogic 本人定期发布工作会话）；
- **oh-my-pi**：社区扩展集合（hash 锚定编辑、LSP、浏览器、子代理等）；
- **OpenClaw**：使用 SDK 嵌入 PI 的真实集成案例（pi.dev 官方举例）；
- **pi.dev/packages**：package 市场（gallery），扩展/技能/模板/主题分发；
- **Gondolin 扩展**：把内置工具与 `!` 命令路由进本地 Linux 微虚拟机做隔离；
- **learning-pi-agent**（社区）：有中文架构笔记，与本文结论一致；
- 网上中文报道称 GitHub star 约 9.3 万（2026-08），未在源码内验证，仅供参考。

## 7.3 建议调研方向

如果继续深入研究，按以下线索展开最有价值：

1. **Provider 抽象与懒加载**（`packages/ai/src/models.ts` + `api/*.lazy.ts`）
   - 研究 `createProvider` 如何用一份定义同时支持静态目录、动态刷新、OAuth/API key、混合 API 分发；
   - 研究 `lazyStream` 的懒加载时机如何影响启动性能与错误传播。
2. **扩展事件模型**（`packages/coding-agent/src/core/extensions/types.ts`）
   - 34 种语义事件的完整时序图（尤其 `before_agent_start`、`tool_call`、`session_before_compact` 三个扩展点）；
   - 这是 PI 生态的"API 面"，写扩展/做集成必读。
3. **Agent 循环的并发与取消语义**（`packages/agent/src/agent-loop.ts`）
   - 并行工具执行的 preflight/execute/finalize 三段如何保证事件顺序与状态一致；
   - AbortSignal 在 LLM 流、工具、队列三处的传播路径。
4. **压缩质量与文件跟踪**（`packages/coding-agent/src/core/compaction/`）
   - `readFiles`/`modifiedFiles` 文件操作追踪如何让摘要"懂代码"；
   - 扩展自定义压缩（主题化/代码感知）的接口形态。
5. **TUI 差分渲染**（`packages/tui`）
   - CSI 2026 原子刷新 + 差分行更新 + Kitty/iTerm2 内联图片协议。
6. **远程协议**（`packages/protocol` + `packages/server`）
   - CBOR 编解码、帧格式、握手/心跳/快照广播——未来多进程/远程 Agent 的调研起点。
7. **AgentHarness 与测试**（`packages/agent/src/harness/` + `packages/evals`）
   - lane/run/compaction/navigation + records 日志 + 崩溃恢复（`reducer.ts`）是下一代 durable harness 的核心，当前仍是脚手架；
   - 已跑通的是"真实产品栈"评估：`pi-evals` 适配 `AgentSession` + vitest-evals（见 [08-eval-testing.md](./08-eval-testing.md)）。

## 7.4 参考链接

- 官网与文档：<https://pi.dev>、<https://pi.dev/docs/latest>
- 源码仓库：<https://github.com/earendil-works/pi>（本仓库即其镜像）
- npm：`@earendil-works/pi-coding-agent` / `@earendil-works/pi-agent-core` / `@earendil-works/pi-ai`
- 社区会话数据集：<https://huggingface.co/datasets/badlogicgames/pi-mono>
- 第三方架构分析：
  - agentic-ai 文档 Pi 章节：<https://agentic-ai.readthedocs.io/en/latest/AgentHarness/pi-dev/>
  - learning-pi-agent 中文笔记：<https://github.com/yamsfeer/learning-pi-agent>
- 容器化：`packages/coding-agent/docs/containerization.md`（Gondolin/Docker/OpenShell 三种模式）

## 7.5 免责声明

本目录为分析性文档，非官方文档；代码路径与行为以本仓库当前快照为准，公开资料数据（star 数、release 数等）仅作背景参考，可能滞后于最新状态。
