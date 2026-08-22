# PI vs Codex CLI vs DeepSeek Harness：三框架对比

> 对比基于三份独立分析文档（`analysis-docs/pi/`、`analysis-docs/codex/`、`analysis-docs/deepseek-harness/`），快照日期 2026-08-22。矩阵单元格引用各自分析文档；"支持 X"以源码为准。

## 1. 执行摘要

| 框架 | 一句话定位 | 最大的一个决定性差异 |
|---|---|---|
| **PI**（pi-agent-harness） | 最小核心 + 强扩展的编码 agent 框架：4 个默认工具、~1000 token 系统提示、TypeScript 扩展组装一切 | **刻意不做**：MCP/子代理/权限弹窗/plan mode 都留给扩展，核心只留"循环 + 4 工具 + 会话" |
| **Codex CLI**（openai/codex） | OpenAI 的单厂商产品化编码 agent：Rust 核心、深度绑定 Responses API 与 ChatGPT 账号 | **安全纵深优先**：AskForApproval 四态 + exec-policy 规则 + 沙箱 + Guardian 自动审批 + MCP 审批桥，权限体系三框架中最重 |
| **DeepSeek Harness**（dsh） | 一切皆插件的组合平台：连模型适配、工具、会话日志、agent 循环本身都是 Cordis 插件，没有特权核心 | **配置树即架构**：profile/bundle/cordis.patch.yml 三层覆盖任何一行，可替换循环本身 |

## 2. 维度矩阵

| 维度 | PI | Codex CLI | dsh |
|---|---|---|---|
| **定位/哲学** | 最小核心 + 扩展（pi 01 §1.2） | 单厂商产品化 CLI（codex 01 §1.2） | 一切皆插件、无特权核心（dsh 01 §1.2） |
| **技术栈** | TypeScript monorepo（9 包，pnpm） | Rust workspace（112 crates，bazel）+ TS/Python SDK（codex 02 §2.2） | TypeScript monorepo（~40 包，Cordis，pnpm）（dsh 02 §2.1） |
| **架构分层** | pi-ai → pi-agent-core → pi-coding-agent 单向（pi 02 §2.1） | core/session 为核心，tui/exec/mcp/app-server 复用（codex 02 §2.1） | 配置树：boot → bundle → core → 能力缝（dsh 02 §2.1） |
| **Agent 循环** | 双层循环：LLM↔工具内循环 + follow-up 外循环；steer/followUp 队列（pi 03 §3.2） | turn 内多 step：采样↔工具循环 + pending input + mid-turn 压缩（codex 03 §3.1） | turn（≥0 step）/ step 双层；inbox next-turn/next-step；4 个瀑布钩子（dsh 03 §3.2） |
| **停止条件** | stopReason（error/aborted/length）+ shouldStopAfterTurn 钩子 | turn-stop hooks（block→继续 / stop）+ token limit 压缩 | turnEndReason（completed/max-tokens/aborted/error/blocked）；max-tokens 粘性 |
| **steer/follow-up** | steer（当前 turn 后注入）+ followUp（停止前注入），all/one-at-a-time（pi 03） | pending input 队列 + mailbox（InterAgentCommunication）（codex 03） | inbox 三通道：followup/steer/inject，next-turn/next-step 边界（dsh 03 §3.2） |
| **工具系统** | 4 默认工具 + 可选 grep/find/ls；prepare→execute→finalize；默认并行（pi 03 §3.3） | 注册表 + 并行 in-flight + 审批/沙箱包裹（codex 03 §3.2-3.4） | 注册表 + 调度器 prepare→dispatch→finalize/finish；并行池上限 10 + 独占屏障（dsh 03 §3.3） |
| **工具 schema** | typebox JSON Schema + prepareArguments 兼容 | JSON Schema + 动态工具 + code-mode 命名空间（codex 05） | JSON Schema/TS/Python 三语生成（dsh 01 §1.3） |
| **权限** | 无逐工具权限；项目信任门禁 + bash 输出清洗（pi 09） | AskForApproval 四态 + exec-policy 规则 + Guardian（codex 07 §7.1） | SandboxMode 三档 + ask 审批 + escalation（dsh 07 §7.1） |
| **沙箱** | 扩展路径（Gondolin/Docker/OpenShell/示例扩展）（pi 09 §9.4） | 内置 bwrap / Windows sandbox + exec-server（codex 07 §7.2） | 内置 sandbox-local（bwrap/sandbox-exec/ACL）+ e2b 远程（dsh 07 §7.1） |
| **Provider 抽象** | Provider/Models 接口 + 45 工厂（pi 06 §6.2） | ModelProvider trait + 内置 OpenAI/ChatGPT/lmstudio/ollama/bedrock（codex 06 §6.2） | LlmAdapter 注册表 + pi-ai 孪生（dsh 06 §6.1-6.2） |
| **认证** | auth.json / OAuth（设备码/PKCE）/ env（pi 06 §6.3） | keyring + ChatGPT 账号登录（codex 06 §6.2） | 凭证引用 env/file/project/user + 每请求解析（dsh 06 §6.2） |
| **上下文工程** | buildSystemPrompt 动态组装 + skills 渐进披露（pi 01/04） | clone_history().for_prompt + skills/plugins 注入 + turn diff（codex 04 §4.3） | PromptSection/PromptContext 有序组装 + system-prompt/assemble 瀑布（dsh 04 §4.4） |
| **会话持久化** | 树形 JSONL v3 + 分支/fork/压缩（pi 04） | rollout JSONL + SQLite state db（codex 04 §4.1） | SessionEvent 追加日志 v0 + 可选 SQLite 搜索（dsh 04 §4.1） |
| **压缩** | 手动/阈值/溢出三触发，摘要 + firstKeptEntryId（pi 04 §4.5） | pre-turn / mid-turn 上下文上限 / 远端压缩（codex 04 §4.4） | compaction 事件 + tool-result pruner（dsh 04 §4.5） |
| **扩展系统** | 两阶段 Loader/Runner + 34 种语义事件（pi 05） | hooks 8 类 + 插件/marketplace + MCP + skills（codex 05） | Cordis 插件树 + scoped 注册 + 瀑布/串行事件（dsh 05） |
| **子代理** | 不内置（扩展实现）（pi 01 §1.2） | 原生：AgentRegistry + 角色 + builtins（codex 05 §5.6） | 原生：ctx.subagents 多 provider + continuable（dsh 05 §5.5） |
| **MCP** | 不内置（扩展实现） | 内置客户端 + 服务端（codex 05 §5.4） | 内置 mcp-client（dsh 01 §1.3） |
| **评估/测试** | pi-evals（vitest-evals）+ session conformance + faux provider（pi 08） | crate 内单测 + bazel/cargo + 大体积集成测试（codex 07 §7.4） | vitest 多配置（unit/e2e/snapshot/web/stress）+ 门禁（dsh 07 §7.3） |
| **运行模式** | interactive/print/json/rpc/sdk + 实验远程（pi 05 §5.4） | tui/exec/review/mcp-server/app-server/agents/cloud（codex 05 §5.7） | web/headless/dump-config/plugin/ACP（dsh 05 §5.4） |
| **许可证** | MIT | Apache-2.0 | MIT |

## 3. 关键差异（决定性 3 项）

### 3.1 扩展深度：从"允许扩展"到"扩展就是本体"

- **PI**：核心固定（循环 + 4 工具 + 会话），扩展是"附加层"；事件/工具/命令/UI 都可扩展，但核心不因扩展而改变（pi 01 §1.2、pi 05）。
- **Codex**：多机制并存（hooks/插件/MCP/skills/子代理），但核心产品（Responses API 绑定、审批沙箱）不可替换——扩展是"围绕固定核心的插口"（codex 05 §5.1）。
- **dsh**：**连核心都可替换**——agent 循环本身是插件（`packages/core/agent-loop/src/index.ts`），模型适配、会话日志、工具全部可经 `cordis.patch.yml` 覆盖；"没有特权核心"是源码事实而非宣传（dsh 01 §1.2、dsh 05 §5.1）。

短 trace：想"给系统提示加一段项目规则"——

- PI：改 `AGENTS.md` 或扩展 `before_agent_start` 事件（pi 05 §5.2）；
- Codex：用 `UserPromptSubmit`/`SessionStart` hook 注入（codex 03 §3.5）；
- dsh：写一个 `PromptSection` 注册到 `ctx.systemPrompt`，或直接在 profile patch 里覆盖 `system-prompt` 行（dsh 04 §4.4、dsh 02 §2.2）。

三者都能做到，但 dsh 是"配置"，pi 是"代码/文件"，codex 是"hook 脚本"。

### 3.2 权限模型：从"无权限"到"安全纵深"

- **PI**：默认以用户权限运行、无逐工具审批；信任门禁只管 `.pi` 资源加载；沙箱是扩展/容器（pi 09）。
- **Codex**：命令先过 exec-policy（规则 + 危险命令启发式）→ `AskForApproval` 四态决策 → Guardian/MCP 审批桥 → 沙箱执行；`--dangerously-bypass-approvals-and-sandbox` 是显式逃生舱（codex 03 §3.4、codex 07）。
- **dsh**：每调用解析 `SandboxPolicy`（read-only/workspace-write/danger-full-access），默认 workspace-write + ask；审批记录为持久化事件可审计；escalation 支持扩大权限重试（dsh 07 §7.1）。

对"模型乱跑 bash"场景：PI 依赖用户自觉/容器；Codex 有完整审批链；dsh 有沙箱 + ask 默认值，但审批策略可通过配置替换。

### 3.3 会话模型的哲学：消息树 vs 事件日志 vs rollout 序列

- **PI**：JSONL v3 消息树（id/parentId），模型上下文 = 叶子回溯（pi 04）；
- **dsh**：**事件日志即事实源**，"模型可见即已落盘"是运行时不变式；UI/回放/压缩/持久化全部从同一日志派生（dsh 04 §4.1）；
- **Codex**：rollout JSONL 记录 ResponseItem 序列 + SQLite 索引；恢复靠 `rollout_reconstruction` 反向重建（codex 04 §4.1）。

影响：dsh 的审计/回放能力最强（一切皆事件）；PI 的分支/fork 最轻量（树指针）；Codex 的 SQLite 索引让"恢复/搜索"最快，但格式无显式版本。

## 4. 选型建议

- **当你要"最小可控、自己组装"时选 PI**：团队能写 TypeScript 扩展，想要短提示词 + 少工具 + 完全本地/自托管（MIT、零 SaaS）。验证假设：你接受自己补 MCP/子代理/权限门。
- **当你深度使用 OpenAI 生态、需要开箱即用与强权限控制时选 Codex**：ChatGPT 账号/Responses API 已是默认，审批+沙箱+Guardian 可直接上线。验证假设：你接受单厂商绑定与 Apache-2.0 下的自有代码量（Rust）。
- **当你想"按配置组装一个 harness"、需要可替换能力与强审计时选 dsh**：一切皆插件 + 事件日志 + 内置子代理/MCP/沙箱。验证假设：你能接受 developer preview（格式 v0 无迁移、破坏性变更频繁）。

## 5. 附录

### 分析文档

- PI：[analysis-docs/pi/README.md](../pi/README.md)（含 01–09）
- Codex：[analysis-docs/codex/README.md](../codex/README.md)（含 01–07）
- dsh：[analysis-docs/deepseek-harness/README.md](../deepseek-harness/README.md)（含 01–07）

### 快照

| 框架 | 分支 | commit | 版本 |
|---|---|---|---|
| pi | main | c49906ec7 | 0.84.2（-84） |
| codex | main | 4f39251a01 | rusty-v8-v150.4.0-1014 |
| dsh | master | b150a551b8 | 0.1.1-rc.2 |

### 注意事项

- 三个仓库均处于快速迭代：dsh 明确 developer preview，codex 快照为开发主线；对比结论会随 commit 漂移，引用时以各自 README 快照为准。
- 部分"特性"是宣传 vs 源码差异：例如 PI 的"系统提示词 ~1000 token"仅部分验证（pi 07 §7.1）；codex 的 star/社区数据未在源码验证；dsh 的"无特权核心"已在源码证实（循环即插件）。
- 工具数、事件数、provider 数均为快照实测（PI 4+3 工具/34 事件/45 工厂；codex 工具在 tools crate；dsh 默认工具集在 base patch），对比时勿跨版本混用。
