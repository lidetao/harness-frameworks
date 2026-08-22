# 3. 运行机制

## 3.1 启动流程

```mermaid
sequenceDiagram
    participant U as 用户/终端
    participant CLI as main.ts
    participant SM as SessionManager
    participant RT as ModelRuntime
    participant RL as ResourceLoader
    participant S as AgentSession
    participant A as "Agent(agent-core)"
    participant P as pi-ai Provider
    participant T as "工具(read/bash/edit/write)"

    U->>CLI: pi "请修复 xxx"
    CLI->>CLI: 解析参数 → 决定模式(interactive/print/rpc)
    CLI->>SM: 新建/继续/恢复/分叉 session
    CLI->>RT: ModelRuntime.create()<br/>发现provider、刷新模型目录
    CLI->>RL: 加载 settings/AGENTS.md/扩展/技能/主题
    CLI->>S: createAgentSession()
    S->>A: new Agent(streamFn=modelRuntime.streamSimple)
    S->>S: 构建系统提示词、注册默认工具、订阅事件
    CLI->>S: session.prompt(text)
    S->>S: 展开技能/模板、校验模型与认证
    S->>A: agent.prompt(messages)
    A->>A: runAgentLoop 开始事件流
    A->>P: streamSimple(model, 上下文, tools)
    P-->>A: 流式事件(text_delta/toolcall_end...)
    A-->>S: message_start/update/end 事件
    S->>SM: message_end → 追加 JSONL 持久化
    Note over A,P: 若含 toolCall
    A->>T: executeToolCalls(prepare→execute→finalize)
    T-->>A: toolResult 消息
    A-->>S: 工具结果事件(持久化/UI更新)
    Note over A,P: 循环直到无工具调用/队列空
    A-->>S: agent_end
    S->>S: 检查重试/自动压缩/队列
    S-->>U: agent_settled，UI恢复空闲
```

启动要点：

- 会话对象（`AgentSession`）在 CLI 解析阶段之后创建，由 `createAgentSessionServices` 组装 `ModelRuntime`/`SettingsManager`/`ResourceLoader`/`SessionManager`；
- 模型选择优先级：CLI 参数（`--model`/`--provider`）> 会话中恢复的模型 > 设置默认 > 第一个可用模型；thinking level 会按模型能力 clamp；
- 已有会话恢复时，`SessionManager.buildSessionContext()` 从叶子回溯重建消息、模型、thinking 状态；
- 系统提示词由 `buildSystemPrompt` 动态组装（工具列表 + 技能 + 项目上下文文件 + 自定义 prompt）。

## 3.2 Agent 双层循环

### 分层设计

```mermaid
flowchart TD
    subgraph 高层["Agent(有状态)"]
        ST[state: systemPrompt/model/thinkingLevel/tools/messages]
        Q[steeringQueue / followUpQueue]
        AB[AbortController]
        EV[事件订阅 listeners]
    end

    subgraph 低层["agent-loop(无状态)"]
        LOOP[runLoop 双嵌套循环]
        STREAM[streamAssistantResponse]
        EXEC[executeToolCalls]
    end

    ST --> LOOP
    Q --> LOOP
    LOOP --> STREAM
    LOOP --> EXEC
    LOOP --> EV
```

低层循环是纯函数式：输入 `AgentContext + AgentLoopConfig`，通过 `emit` 回调输出 `AgentEvent`，不持有可变状态。高层 `Agent` 持有 transcript 和队列，负责把事件应用到状态并转发给订阅者。

### runLoop 结构

```mermaid
flowchart TD
    START["runAgentLoop<br/>发射 agent_start / turn_start<br/>注入 prompts"] --> LOOP["runLoop"]
    LOOP --> PENDING{"pendingMessages<br/>(steering) 有消息?"}
    PENDING -->|是| INJECT["注入消息<br/>message_start/message_end"]
    PENDING -->|否| LLM["streamAssistantResponse<br/>transformContext → convertToLlm<br/>→ provider.streamSimple"]
    INJECT --> LLM
    LLM --> MSG["assistant消息"]
    MSG --> ERR{"stopReason<br/>error/aborted?"}
    ERR -->|是| END["agent_end"]
    ERR -->|否| TOOLCALL{"含toolCall?"}
    TOOLCALL -->|是| EXEC["executeToolCalls<br/>串行/并行"]
    EXEC --> RESULTS["toolResult消息<br/>回灌上下文"]
    TOOLCALL -->|否| TURNEND["turn_end"]
    RESULTS --> TURNEND
    TURNEND --> NEXT["prepareNextTurn<br/>(可换模型/上下文)"]
    NEXT --> STOP{"shouldStopAfterTurn?"}
    STOP -->|是| END
    STOP -->|否| STEER["轮询 steering 队列"]
    STEER --> PENDING
    PENDING -->|否| FOLLOW{"followUp 队列?"}
    FOLLOW -->|是| INJECT
    FOLLOW -->|否| END
```

要点：

- **内循环**：一次 LLM 调用 + 该轮全部工具执行为一个"turn"；只要还有 toolCall 或 steering 消息就继续；
- **外循环**：内循环结束后检查 follow-up 队列，有则注入继续，否则 `agent_end`；
- **steering**：用户运行中输入的消息，在当前回合工具执行完后、下一次 LLM 调用前注入（`steer()` 入队，`getSteeringMessages` 轮询）；队列模式支持 `all`/`one-at-a-time`；
- **follow-up**：用户希望等 Agent 跑完再处理的消息（Alt+Enter），在 Agent 即将停止时注入；
- **钩子**：`shouldStopAfterTurn`（优雅停）、`prepareNextTurn`（换模型/thinking/上下文）、`transformContext`（上下文裁剪/注入）、`convertToLlm`（AgentMessage → LLM Message）；
- **错误契约**：`StreamFn` 不允许抛异常，失败必须编码在流内（`stopReason: "error"/"aborted"` + `errorMessage`），保证循环可继续处理。

源码对应（`packages/agent/src/agent-loop.ts`）：

| 结论 | 位置 |
|---|---|
| 外循环（follow-up 队列驱动 `while(true)`） | `agent-loop.ts:155`（`runLoop`） |
| 内循环（toolCall / steering 驱动） | `agent-loop.ts:174` |
| 队列注入点（注入后进入下一次 LLM 调用） | `agent-loop.ts:182-190` |
| `stopReason: error/aborted` → `agent_end` | `agent-loop.ts:196` |
| `stopReason: "length"` → 整批 toolCall 失败 | `agent-loop.ts:212-215`、`381` |
| turn 间 `prepareNextTurn` / `shouldStopAfterTurn` | `agent-loop.ts:232`、`248` |
| steering / follow-up 轮询回调 | `agent-loop.ts:259`、`263` |
| LLM 调用边界（transformContext → convertToLlm → streamFn） | `agent-loop.ts:281-327` |
| 流式事件转发（start/text_delta/toolcall_end/done/error） | `agent-loop.ts:319-368` |

高层 `Agent`（`packages/agent/src/agent.ts`）：`steer()`/`followUp()` 入队（`agent.ts:283`/`288`），队列默认 `one-at-a-time`（`agent.ts:231-232`），`abort()` 通过 `AbortController` 传播（`agent.ts:319-320`），`agent_end` 后等所有监听器落定才算 idle（`agent.ts:328`）。重复 `prompt()` 会抛错，要求改用 `steer`/`followUp`（`agent.ts:350-354`）。

## 3.3 工具执行管道

```mermaid
flowchart LR
    A["assistant消息中的toolCall"] --> B["Prepare<br/>查找工具定义<br/>prepareArguments 归一化<br/>JSON Schema 校验<br/>beforeToolCall 钩子"]
    B -->|"拦截/校验失败/abort"| E1["立即生成错误toolResult"]
    B -->|通过| C{"执行模式"}
    C -->|"sequential"| D1["逐个执行 tool.execute"]
    C -->|"parallel"| D2["并发执行允许的工具"]
    D1 --> F["Finalize<br/>afterToolCall 钩子<br/>可覆盖 content/isError/usage/terminate"]
    D2 --> F
    F --> G["toolResult消息<br/>追加上下文 + 持久化 + UI"]
    E1 --> G
```

细节：

- **Prepare 阶段**：按名字查找工具；`prepareArguments` 做参数兼容归一化；`validateToolArguments` 做 JSON Schema 校验；`beforeToolCall` 可 `block`（扩展用它实现权限门）并设置 `terminate` 参与整批提前终止；
- **Execute 阶段**：默认 `parallel`——先逐个 preflight（prepare），再并发执行；执行中可通过 `onUpdate` 回调流式上报部分结果（如 bash 输出）；`executionMode: "sequential"` 的工具强制串行；
- **Finalize 阶段**：`afterToolCall` 可整体替换结果内容、错误标志、usage；`terminate` 仅当**整批**所有工具结果都置 true 才提前终止；
- **截断保护**：`stopReason === "length"` 时，整批 toolCall 判定为失败（参数可能被截断），提示模型重新发起，绝不执行半截参数；
- **错误处理**：工具抛异常不打断循环，转为 `isError: true` 的 toolResult 回灌给模型，模型可自纠。

源码对应（同文件）：`executeToolCalls` 判定 sequential/parallel（`agent-loop.ts:411`）；串行（`433`）/并行（`489`）；`prepareToolCall`（`600`，含 `beforeToolCall` block 与 abort 检查）；`executePreparedToolCall`（`670`，onUpdate 事件在 execute settle 后统一 flush）；`finalizeExecutedToolCall`（`713`，afterToolCall 字段级覆盖）；`shouldTerminateToolBatch` 全批 terminate（`582`）。

## 3.4 事件系统

Agent 事件（`AgentEvent`，订阅在 `agent.subscribe`）：

| 事件 | 语义 |
|---|---|
| `agent_start` / `agent_end` | 一次 run 的生命周期 |
| `turn_start` / `turn_end` | 一个 turn = 一次 LLM 调用 + 其工具执行 |
| `message_start` / `message_update` / `message_end` | 消息生命周期（user/assistant/toolResult；`update` 仅流式 assistant） |
| `tool_execution_start` / `tool_execution_update` / `tool_execution_end` | 工具执行生命周期 |

AgentSession 扩展事件（`AgentSessionEvent`）：`agent_settled`、`queue_update`、`compaction_start/end`、`entry_appended`、`thinking_level_changed`、`auto_retry_start/end`、`bash_execution_update`、`summarization_retry_*` 等。

事件分发顺序（一次 message_end）：

```mermaid
sequenceDiagram
    participant A as "Agent(agent-core)"
    participant S as AgentSession
    participant E as ExtensionRunner
    participant UI as "模式层(interactive/rpc)"
    participant SM as SessionManager

    A->>S: message_end
    S->>E: 先转发给扩展(message_end)
    E-->>S: 可能返回替换消息
    S->>UI: 通知会话监听者
    S->>SM: appendMessage 持久化
    Note over S: 扩展可在 message_end 替换消息内容<br/>(replaceMessageInPlace 保持状态一致)
```

`agent_end` 之后 AgentSession 还会执行 `_handlePostAgentRun`：判断自动重试、自动压缩、是否有队列消息需要 `agent.continue()`，最后发射 `agent_settled`（UI 恢复空闲）。

## 3.5 新一代 durable harness（agent-core，脚手架）

`packages/agent/src/harness/` 是正在成型的新一代 harness 运行时，与 3.1-3.4 的"当前产品栈"并行存在。它把 Agent 运行升级为**可崩溃恢复的耐用操作**：

```mermaid
flowchart TD
    APP[宿主应用] --> H["AgentHarness<br/>(AgentLane)"]
    H -->|prompt/skill/template| RUN[run 操作]
    H -->|compact| CMP[compaction 操作]
    H -->|navigateTree| NAV[navigation 操作]
    RUN --> RED["reducer 状态机<br/>(per-lane)"]
    CMP --> RED
    NAV --> RED
    RED -->|操作步骤| R1["step_attempt<br/>assistant/compaction/branch_summary"]
    RED -->|工具| R2["tool_started<br/>replay: never/safe"]
    RED -->|队列| R3["queue_enqueued<br/>steer/followUp/nextRun"]
    RED -->|异步| R4["write_deferred<br/>DeferredHandle"]
    R1 --> SESSION["SessionRepo<br/>entries + records 持久化"]
    R2 --> SESSION
    R3 --> SESSION
    R4 --> SESSION
    SESSION --> B1["JSONL v4"]
    SESSION --> B2["memory"]
    SESSION --> B3["SQLite (writer leases/branch cache/search)"]
```

要点（均为本快照源码）：

- **操作模型**：lane 上三类操作——`run`（prompt/skill/template）、`compaction`、`navigation`；结果类型为 `RunOutcome`/`CompactionOutcome`/`NavigationOutcome`（completed/aborted/failed/suspended，`agent-harness.ts:89`）。`AgentLane` 接口还含 `resume`、`abort`、`steer`/`followUp`/`nextRun` 三队列、`recordUsage`、`watch`（`agent-harness.ts:271`）。
- **records 日志**：会话是 entries（消息/模型/工具/压缩等内容）与 records（操作状态：`operation_started`/`abort_requested`/`step_attempt`/`tool_started`/`queue_enqueued`/`queue_cancelled`/`write_deferred`/`operation_finished`，`packages/agent/src/harness/session/types.ts`）双写；单写者 seq 协议保证顺序（`session/state.ts` 的 `applyMutation` 校验非连续 seq 即拒绝）。
- **崩溃恢复**：`resume()` 读取未完成操作与记录日志重建 lane 状态；`RecordLogCorruption`（`reducer.ts`）用机器可读原因（如 `multiple_open_operations`、`non_consecutive_attempt`、`duplicate_tool_invocation`）拒绝无法自洽的日志切片，而不是"尽力修复"。
- **工具可重放性**：`HarnessTool = AgentTool & { replay?: "never" | "safe" }`（`agent-harness.ts:237`）——`replay: "safe"` 的工具（如只读）在恢复时可安全重放，`"never"`（如写文件）则必须跳过。
- **hooks/events**：`before_run`/`before_resume`/`before_run_end`/`transform_context`/`before_request`/`before_payload`/`after_response`/`before_tool`/`after_tool`/`before_compaction`/`before_navigation`（`agent-harness.ts:198` HookName）；`drive: "automatic" | "manual"` 决定是否由内部推进，manual 下宿主用 `peekAction`/`executeAction` 逐步驱动。
- **当前状态**：`AgentHarness` 构造时可用，但 `prompt`/`skill`/`compact`/`navigateTree` 等方法统一抛 `HarnessNotImplemented`（`agent-harness.ts:355` 的 `unavailable`）；`createCodingAgentHarness`（`packages/coding-agent/src/server/create-harness.ts`）已把 coding-agent 工具与系统提示词接好，等待运行时补齐。
- **测试先行**：reducer、compaction、branch-summarization、session 后端均有专项测试，SQLite 后端跑同一套 conformance（详见 [07-security-eval.md](./07-security-eval.md)）。
