# 6. LLM 提供商抽象

## 6.1 ModelProvider trait

`codex-rs/model-provider/src/provider.rs:141`：

```rust
pub trait ModelProvider: fmt::Debug + Send + Sync {
    // 认证/模型解析/流式采样/远端压缩能力…
}
```

- 运行时入口 `SharedModelProvider`（注入 `Session.services.provider`）；每个 turn 的 `TurnContext` 持有 provider 信息（`turn_context.rs`）；
- `ModelProviderInfo`（`model-provider-info/src/lib.rs:96`）声明 provider 属性（name、requires_openai_auth、远程压缩支持 `RemoteCompactionSupport` 等）；`create_openai_provider`（`lib.rs:385`）构造默认 OpenAI provider；
- 认证解析：`ResolvedProviderAuth`（`model-provider/src/auth.rs:53`）+ `AuthProvider` trait（AgentIdentity/Header/AuthManager/Unauthenticated 四实现，`auth.rs:87-176`）。

## 6.2 内置提供商

| Provider | 说明 |
|---|---|
| OpenAI（默认） | Responses API（`codex-rs/tools/src/responses_api.rs`），支持 WebSocket 实时（`client.rs` 的 `responses_websocket_enabled`/preconnect） |
| ChatGPT 账号 | `codex login` 走 OpenAI 账号（Plus/Pro/Team），`codex-login` crate；`login.rs`/`logout` 子命令 |
| LM Studio / Ollama | 本地模型（`lmstudio/`、`ollama/` crate） |
| Amazon Bedrock | `model-provider/src/providers/amazon_bedrock` |
| Azure OpenAI | `model-provider-info` 的 `create_azure_provider`（`lib.rs:545` 附近配置化） |

## 6.3 模型目录与配置

- 模型配置在 `config.toml` 的 `[model_providers.<id>]`（base_url/api_key_env/models），`ModelProviderInfo::from_models_toml` 类路径解析（`model-provider-info/src/lib.rs:545-562`：built-in provider 与配置 provider 合并）；
- `models-manager` crate 管理模型列表；每次 turn 解析 exact model（`turn_context` 的 `model_info`），`reasoning_summary`/`service_tier` 一并解析（`turn.rs:2227-2240`）；
- 远端压缩：OpenAI provider 支持 `compact_remote_v2`（`compact_remote_v2.rs`），把压缩委托给服务端。

## 6.4 流式协议与重试

- `ModelClientSession.stream`（`core/src/client.rs:1883`）返回流，turn 内复用（含 WebSocket/sticky 路由状态）；
- 重试：`responses_retry.rs`（`ResponsesStreamRetryState`、`handle_retryable_response_stream_error`），turn 内 client_session 复用；
- 事件：`ResponseEvent`（`client_common.rs`）→ `ResponseItem` 流；`codex_protocol` 定义完整响应项模型（`models.rs`，含 reasoning/text/tool call 等）。

## 6.5 与 pi/dsh 对比（快照级）

| 维度 | codex | pi | dsh |
|---|---|---|---|
| 抽象 | `ModelProvider` trait + 内置集 | `Provider`/`Models`（models.ts:97/156） | `LlmAdapter` 注册表（llm/index.ts:365） |
| 默认认证 | ChatGPT 账号 / OpenAI API key | 45+ provider 自带 key/OAuth | deepseek-official + pi-ai 孪生 |
| 模型目录 | config.toml + ModelProviderInfo | 生成式静态目录 + 动态刷新 | 发现式（LlmDiscoveredModel） |
| 流协议 | Responses API + WebSocket | 统一 AssistantMessageEventStream | StreamChunk + BlockAssembler |
| 远端压缩 | 有（compact_remote_v2） | 无（本地摘要） | 无（本地 compaction 事件） |
