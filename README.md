# Harness Frameworks 分析文档

对开源 AI agent / harness 框架的源码分析集合。每个框架一套独立分析文档（基于 `analyze-harness-framework` 技能：Agent 循环、工具系统、provider 抽象、上下文工程、会话与记忆、扩展系统、评估、安全/权限、运行模式、设计哲学），并附横向对比。**所有结论均以源码为准，关键结论附 `path:line` 证据。**

## 目录

| 项目 | 定位 | 快照 | 文档 |
|---|---|---|---|
| [pi](./pi/README.md) | 最小核心 + 强扩展的编码 agent 框架（TypeScript） | main `c49906ec7` / 0.84.2 | 01 核心功能 · 02 架构 · 03 运行时 · 04 会话 · 05 扩展 · 06 Provider · 07 调研 · 08 评估 · 09 安全 |
| [codex](./codex/README.md) | OpenAI 的单厂商产品化编码 agent（Rust） | main `4f39251a01` | 01 核心功能 · 02 架构 · 03 运行时 · 04 会话 · 05 扩展 · 06 Provider · 07 安全与评估 |
| [deepseek-harness](./deepseek-harness/README.md) | 一切皆插件的 agent harness 组合平台（Cordis/TypeScript） | master `b150a551b8` / 0.1.1-rc.2 | 01 核心功能 · 02 架构 · 03 运行时 · 04 会话 · 05 扩展 · 06 Provider · 07 安全与评估 |
| [三方对比](./comparisons/pi-vs-codex-vs-dsh.md) | pi vs Codex CLI vs dsh | — | 维度矩阵 · 关键差异 · 选型建议 |

## 阅读建议

- **只想了解一个框架**：直接看该项目 README 的"核心结论（TL;DR）"，再按需进入 01–07/09。
- **想选型**：先读 [三方对比](./comparisons/pi-vs-codex-vs-dsh.md) 的执行摘要与选型建议，再回到各框架文档核验。
- **想验证结论**：文档内所有 `path:line` 均相对对应源码仓库根（`../pi`、`../codex`、`../deepseek-harness`），可直接对照。

## 三框架一句话差异

- **pi**：刻意不做——MCP/子代理/权限弹窗/plan mode 全留给 TypeScript 扩展，核心只留"循环 + 4 工具 + 树形 JSONL 会话"。
- **codex**：安全纵深优先——AskForApproval 四态 + exec-policy 规则 + 沙箱 + Guardian 自动审批，深度绑定 OpenAI Responses API。
- **deepseek-harness**：配置树即架构——profile/bundle/cordis.patch.yml 三层覆盖任何一行，连 agent 循环本身都是可替换插件，SessionEvent 事件日志是唯一事实源。

## 文档约定

- 代码路径均相对各源码仓库根；Mermaid 图在 GitHub 上直接渲染。
- 网上资料（star 数、生态等）单独标注在 `07-research.md`（pi）等文件中，不作为源码证据。
- 快照日期 2026-08-22；三个仓库均快速迭代，结论会随 commit 漂移。

## License

本目录分析文档采用与工作区一致的 [LICENSE](./LICENSE)。
