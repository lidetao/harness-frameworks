# 6. LLM 提供商抽象

## 6.1 适配层接口

`packages/llm/llm/src/index.ts`：

```ts
registerAdapter(providers: string[], adapter: LlmAdapter): AdapterRegistrationHandle  // :365
```

- `LlmAdapter`：`prepareCall(provider, model, signal)`（解析 exact-model 默认值/上下文窗口/retry policy，`index.ts:247`）+ `stream(options)`（`AsyncIterable<StreamChunk>`，`index.ts:259`）；
- `LlmRuntime.stream()` 归一化失败：adapter 可以 throw，但流入口把失败转成结构化 `LlmFailure`（`types.ts:309` 注释 + `adapter-failure.ts`），与 dsh 的"错误必须结构化"哲学一致；
- `StreamChunk`（`types.ts:312`）统一增量事件；`BlockAssembler`（`llm/src/assembler.ts`）把 chunk 聚合为消息块，支持 interrupted/usage/replayState（`agent.ts` step 内使用）；
- `llm/stream` 事件是 waterfall：监听者可改写请求或注入中间件（`docs/architecture.md`）。

## 6.2 认证与模型目录

- **凭证是引用**：`credentials/credentials`（`src/index.ts:3-118`）定义 `credentialKey(scope, id)` 命名空间与分层 source：`env` / `file` / `project-env` / `user-env`，管理文档 `$DSH_HOME/.credentials.yaml`；adapter **每次请求时解析**引用（`cordis.patch.yml` 注释："Adapters resolve references per request"），避免长工具执行期间过期。
- **模型发现**：`LlmModelDiscoveryRequest`/`LlmDiscoveredModel`（`types.ts:195/221`），设置页/模型选择器据此列出可用模型。
- **pi-ai 孪生**：`llm/llm-pi-ai`（`src/adapter.ts`、`catalog.ts`、`auth.ts`、`provider.ts`、`discovery.ts`）把 pi-ai 的多提供商栈（OpenAI/Anthropic/Google/Bedrock…）作为 dsh 的一个 adapter 挂载；base bundle 中该行**默认休眠**（零路由），用户在 `llm-pi-ai:` 设置段配置 provider profile 后路由才激活，清空即卸载（`packages/bundle/base/cordis.patch.yml`）。
- **DeepSeek 官方路由**：默认 agent 模型 `deepseek-official/deepseek-v4-flash`（base patch），由 `llm-deepseek` 适配。

## 6.3 重试与 token 计量

- `llm-retry`：`LlmRetryPolicy`/`ResolvedRetryPolicy`，按 provider 注册；会话事件 `llm/retry`、`llm/retry-started`（`known-event-types.ts`）记录重试事实；
- `token-meter`：usage projection、估算、client 侧计量（`packages/llm/token-meter/`）；
- 每次请求的 `request/header`（provider/model/参数/tools 快照）与 `request/context`（contextWindow）都是持久化事件（`agent.ts` buildRequest，`agent.ts:426-515`），resume 时恢复 exact-model 参数。

## 6.4 与 pi / codex 的 provider 层对比（快照级）

| 维度 | dsh | pi | codex |
|---|---|---|---|
| provider 抽象 | `LlmAdapter` 注册表（多 provider 并存） | `Provider` 接口（models.ts:97）+ `Models` 集合 | `ModelProvider` trait（provider.rs:141），built-in 集 |
| 默认 provider | deepseek-official（另挂 pi-ai 孪生） | 45+ 工厂 | OpenAI（+ChatGPT 登录/lmstudio/ollama/bedrock/azure） |
| 认证 | 凭证引用（env/file/project/user）+ 每请求解析 | auth.json / OAuth / env | codex-login（API key / ChatGPT 账号 / OAuth-like 会话） |
| 模型目录 | 发现式（LlmDiscoveredModel） | 生成式静态目录 + 动态刷新 | models.toml（ModelProviderInfo，`model-provider-info/src/lib.rs:96`） |
