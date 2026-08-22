# PI 项目源码分析（analysis-docs/pi）

> 本目录是 PI（Pi Agent Harness）仓库的源码分析文档，基于 2026-08-22 的 `main` 分支快照（commit c49906ec7，所有包版本 0.84.2）整理，并结合 pi.dev 等公开资料交叉验证。

## 阅读路径

| 文档 | 内容 |
|---|---|
| [01-core-functionality.md](./01-core-functionality.md) | 项目定位、核心功能、设计哲学 |
| [02-architecture.md](./02-architecture.md) | 包结构、三层分层、依赖关系、关键类 |
| [03-runtime.md](./03-runtime.md) | 运行机制：启动流程、Agent 双层循环、工具执行管道、事件系统 |
| [04-session.md](./04-session.md) | 会话持久化（树形 JSONL）、分支/fork、压缩、自动重试 |
| [05-extensibility.md](./05-extensibility.md) | 扩展系统、skills/模板/主题/包、四种运行模式 |
| [06-ai-provider.md](./06-ai-provider.md) | pi-ai 提供商抽象、认证、模型目录、流式选项 |
| [07-security-eval.md](./07-security-eval.md) | 安全与权限（信任门禁/沙箱/凭证/供应链）+ 评估体系（pi-evals/conformance/faux provider）+ 网上资料附录 |

## 核心结论（TL;DR）

- PI 是"最小核心 + 强扩展"的编码 Agent harness：默认 4 个工具（`read`/`write`/`edit`/`bash`，另有可选的 `grep`/`find`/`ls`），系统提示词控制在 ~1000 token 以内。
- 三层架构：`pi-ai`（统一调模型）→ `pi-agent-core`（Agent 运行循环）→ `pi-coding-agent`（产品化：会话、工具、扩展、压缩、UI）。
- Agent 循环是双嵌套循环：内循环"LLM → 工具 → 结果回灌 → LLM"，外循环处理 follow-up 队列；steering 队列支持运行中干预。
- 会话是追加式树形 JSONL，支持分支、fork、压缩（`compaction` 条目用 `firstKeptEntryId` 标记保留起点）。
- **两套会话体系并存**：产品层 `SessionManager`（coding-agent，JSONL v3，当前 CLI/UI 实际使用）；新一代 durable harness 的 `Session`/`SessionRepo`（agent-core，JSONL v4 / memory / SQLite 后端 + records 日志，面向崩溃恢复与多后端一致性，本快照中 `AgentHarness` 主体仍是脚手架）。
- 故意不做 MCP、子代理、权限弹窗、plan mode、to-do、后台 bash，全部留给 TypeScript 扩展系统按需补齐。
- 安全面不是"零权限"：默认无逐工具权限弹窗，但**有项目信任门禁**（信任后才加载 `.pi` settings/扩展/skills 等资源），bash 输出清洗/截断，沙箱（Gondolin/Docker/OpenShell/扩展）是官方推荐隔离路径。
- 四种运行模式：interactive（TUI）、print/json（脚本）、rpc（stdin/stdout JSONL）、sdk（Node 嵌入）；另有实验性的 CBOR + Unix socket 远程协议（`pi-protocol`/`pi-client`/`pi-server`）。

## 分析快照

- 仓库：`../pi`（earendil-works/pi 镜像），分支 `main`，commit `c49906ec7`，`v0.84.2-84-gc49906ec7`。
- 关键结论尽量附 `path:line`（相对 `../pi`）；网上资料结论单独标注在 [07-security-eval.md](./07-security-eval.md) 附录 A，不作源码证据。

## 文档约定

- 代码路径均为相对 pi 源码仓库根（本目录 ../pi/）的路径，例如 `packages/agent/src/agent-loop.ts`。
- Mermaid 图在 GitHub 上可直接渲染。
- 文中"当前版本"指本仓库快照（0.84.2），与网上最新 release 可能有差异。
