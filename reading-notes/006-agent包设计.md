# 002 - Agent 包设计详解

## 定位

`@earendil-works/pi-agent-core`（`packages/agent`）是 Pi 的**有状态智能体运行时**，位于 `pi-ai` 和 `pi-coding-agent` 之间。它提供两个抽象层级：

| 层 | 文件 | 职责 |
|----|------|------|
| **底层循环** | `agent.ts` + `agent-loop.ts` | 带事件的 LLM 交互循环：prompt → assistant → tool calls → tool results → loop |
| **上层脚手架** | `harness/agent-harness.ts` | 在底层循环上加 session 持久化、compaction、hooks、技能、队列消息 |

`coding-agent` 只使用 Harness 层；底层 `Agent` 类暴露给需要自定义集成的 SDK 使用者。

---

## 一、底层：Agent + Agent Loop

### 1.1 核心类型

所有类型定义在 `src/types.ts`。

**AgentMessage** — 可扩展的消息联合类型：
```typescript
type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```
`Message` 来自 `pi-ai`（user/assistant/toolResult）。应用通过 **declaration merging** 扩展 `CustomAgentMessages` 接口来添加自定义消息类型（如 `bashExecution`、`custom`、`compactionSummary`），由 `harness/messages.ts` 实现。

**AgentTool** — 扩展了 `pi-ai` 的 `Tool`：
```typescript
interface AgentTool<TParameters, TDetails> extends Tool<TParameters> {
  label: string;                                          // UI 显示名
  prepareArguments?: (args: unknown) => Static<TParameters>;  // 参数预处理
  execute: (id, params, signal?, onUpdate?) => Promise<AgentToolResult<TDetails>>;  // 执行，抛异常表示失败
  executionMode?: "sequential" | "parallel";              // 单工具执行策略覆盖
}
```
Tool 执行时可以通过 `onUpdate` 回调流式发出部分结果（`tool_execution_update` 事件），适合大文件读写等长时间操作。

**AgentEvent** — 循环中产生的事件分为四个生命周期：
```
agent_start → turn_start → message_start/update/end → tool_execution_start/update/end → turn_end → agent_end
```

**StreamFn** — 与 `pi-ai.streamSimple` 同签名但增强契约：失败不抛异常，通过 stream 内 `stopReason: "error"/"aborted"` 传递。这使 Agent 循环在 LLM 调用失败时仍能发出规范的错误事件。

**AgentLoopConfig** — 控制器口，贯穿整个循环：
| 配置项 | 作用 |
|--------|------|
| `convertToLlm` | 将 `AgentMessage[]` 转为 LLM 可理解的 `Message[]`，过滤无法转换的消息 |
| `transformContext` | LLM 调用前对消息做变换（压缩、注入额外上下文） |
| `getApiKey` | 每次 LLM 调用动态解析 API key（支持短期 OAuth token） |
| `shouldStopAfterTurn` | 每轮结束后决定是否优雅退出 |
| `prepareNextTurn` | 每轮结束后更新 model/thinkingLevel/context |
| `getSteeringMessages` | 在 agent 工作时注入"驾驶"消息（不打断当前工具） |
| `getFollowUpMessages` | agent 本来要停止时注入后续消息（等工具完成后才处理） |
| `beforeToolCall` / `afterToolCall` | 工具执行前后的 hook |
| `toolExecution` | `"parallel"`（默认）或 `"sequential"` |

**两条消息队列**（QueueMode 控制一次性取全部还是逐条消费）：
- **steering queue**：当 agent 正在运行时注入，在当前工具批处理完成后插入
- **follow-up queue**：当 agent 完成所有工具调用后才消费，继续下一轮

**AgentContext** — 传入循环的快照：
```typescript
{ systemPrompt: string; messages: AgentMessage[]; tools?: AgentTool[] }
```

### 1.2 Agent 类（`agent.ts`）

`Agent` 是有状态的包装器，持有当前 transcript，暴露 `prompt()`、`continue()`、`steer()`、`followUp()` 等 API。

**核心职责：**
1. **状态管理**：`systemPrompt`、`messages`、`model`、`thinkingLevel`、`tools`（均通过 getter/setter 做防御性拷贝）
2. **运行生命周期**：`createMutableAgentState` 创建可变内部状态，保证任何时候只有一条 `activeRun`
3. **事件分发**：`subscribe()` 注册监听器，所有事件和当前的 abort signal 一起传给监听器；`agent_end` 事件发送后监听器才算完（`waitForIdle()` 等待）
4. **队列管理**：`steer()` / `followUp()` / `clearAllQueues()` 操作 PendingMessageQueue

**运行流程（`prompt()` 为例）：**
```
prompt(text)
  → normalizePromptInput (字符串转 AgentMessage[])
  → runPromptMessages
    → runWithLifecycle (设置 activeRun, AbortController)
      → runAgentLoop(messages, contextSnapshot, loopConfig, emit, signal, streamFn)
        → emit agent_start, turn_start, message_start/end for prompts
        → runLoop (外层: follow-up 重新进入; 内层: steering + assistant + tools)
          → streamAssistantResponse (调用 LLM，转换消息，流式 emit)
          → executeToolCalls (依赖 toolExecution 模式)
          → turn_end → shouldStopAfterTurn? → getSteeringMessages?
        → agent_end
      → activeRun.resolve()
```

**关键设计：**
- `convertToLlm` 默认过滤掉 role 不是 user/assistant/toolResult 的消息
- `streamingMessage` 在 streaming 期间是部分 assistant 消息，message_end 后清除
- `pendingToolCalls` 是只读 Set，UI 可用来展示执行中的工具
- `abort()` 终止当前 run 的 abort controller

### 1.3 Agent Loop（`agent-loop.ts`）

**两个入口：**
- `runAgentLoop(prompts, context, config, emit, signal, streamFn)` — 带新消息启动
- `runAgentLoopContinue(context, config, emit, signal, streamFn)` — 从现有 context 继续（最后一条消息必须是 user/toolResult）

**双循环结构：**
```
外层循环 — 由 follow-up 消息驱动重新进入
  内层循环 — 由 steering 消息和 tool calls 驱动
    turn_start
    处理 pendingMessages（steering/follow-up）
    streamAssistantResponse → 获取 assistant 消息
    executeToolCalls → 获取 tool result 消息
    turn_end
    prepareNextTurn → 更新 model/thinkingLevel
    shouldStopAfterTurn? → 退出
    getSteeringMessages → 继续内循环
  getFollowUpMessages → 继续外循环 或 agent_end
```

**工具执行模式：**

`sequential`：逐个执行，每个工具 complete 后立即 emit `tool_execution_end` 和 toolResult message
```
emit tool_execution_start → prepare → execute → finalize → emit tool_execution_end → emit toolResult message
```

`parallel`：预检查所有工具，然后并发执行允许的工具
```
for each: emit tool_execution_start → prepare
  → 不允许的: emit tool_execution_end (error)
  → 允许的: 加入并发池
for each (并发): execute → finalize → emit tool_execution_end
for each (assistant 原始顺序): emit toolResult message
```

**工具执行流水线：**
```
prepareToolCall:
  1. 查找 tool（未找到 → immediate error）
  2. prepareArguments 预处理参数
  3. validateToolArguments（schema 验证）
  4. beforeToolCall hook（可 block）
  → PreparedToolCall 或 ImmediateToolCallOutcome

executePreparedToolCall:
  调用 tool.execute，捕获 onUpdate → tool_execution_update 事件
  抛异常转 error result

finalizeExecutedToolCall:
  afterToolCall hook 可覆盖 content/details/isError/terminate
```

**terminate 语义：** 当批量工具结果全部设置了 `terminate: true` 时，当前批处理完成后立即 `agent_end`，不检查 shouldStopAfterTurn/steering/followUp。

---

## 二、上层：AgentHarness

`AgentHarness` 是 `coding-agent` 使用的主类，给底层循环增加了会话、压缩、钩子、技能等企业级功能。

### 2.1 核心概念

**Session** — 持久化的会话树，每轮对话追加 entry（message、model_change、compaction、branch_summary 等），支持分支（fork）和导航（navigateTree）。详见 2.6 节。

**ExecutionEnv** — 文件和执行的抽象接口。`FileSystem` + `Shell` 两个 trait，使 agent 可以在 Node.js、浏览器或沙箱中运行。关键方法：`readTextFile`、`writeFile`、`exec`。

**Phase** — Harness 状态机：`idle → turn → idle`（compaction/branch_summary 也阻塞 idle）

**Hooks/事件** — 两种 watch 方式：
- `.subscribe(listener)` — 监听所有事件（AgentEvent + Harness 自有事件）
- `.on(type, handler)` — 监听特定 hook 事件，handler 可返回修改值

**Hook 调用链（LLM 请求生命周期）：**
```
before_agent_start   → 可修改 prompt、systemPrompt、注入额外消息
context              → 可修改发送给 LLM 的消息列表
before_provider_request → 可修改 streamOptions（headers/metadata/transport）
before_provider_payload → 可修改原始请求 payload
after_provider_response → 观察响应状态和 headers
tool_call            → 可 block 工具执行
tool_result          → 可覆盖工具结果
save_point           → turn_end 后持久化完成
settled              → 完全 idle，携带 nextTurn 队列长度
```

### 2.2 消息系统（`harness/messages.ts`）

通过 `CustomAgentMessages` 声明合并添加四种自定义消息：

| 消息角色 | 用途 | LLM 转换 |
|----------|------|----------|
| `bashExecution` | 记录执行的 shell 命令和输出 | 转 user 消息（含命令、输出、退出码） |
| `custom` | 通用自定义消息 | 转 user 消息 |
| `branchSummary` | 分支切换摘要 | 转 user 消息，包裹在 `<summary>` 标签中 |
| `compactionSummary` | 对话压缩摘要 | 转 user 消息，包裹在 `<summary>` 标签中 |

`convertToLlm()` — 遍历 AgentMessage[]，根据 role 进行转换，BashExecution 支持 `excludeFromContext`（不计入 LLM 上下文但保留在会话中）。

### 2.3 技能系统（`harness/skills.ts`）

Skill 定义：
```typescript
interface Skill { name, description, content, filePath, disableModelInvocation? }
```

**加载流程：**
1. 递归扫描目录，找到 `SKILL.md` 文件
2. 解析 YAML frontmatter（name、description、disable-model-invocation）
3. 验证 name（必须全小写+数字+连字符，与父目录同名，不超过64字符）
4. 支持 `.gitignore` / `.ignore` / `.fdignore` 规则
5. name 默认使用目录名（如果 frontmatter 未指定）
6. 返回 `{ skills, diagnostics }`

**技能注入模型的方式：**
- `formatSkillsForSystemPrompt()` — 生成 XML 格式的 available_skills 块，写入 system prompt
- 模型看到技能列表后可自行决定是否"读取技能文件"
- `formatSkillInvocation()` — 通过 `harness.skill(name)` 显式调用，直接注入完整技能内容

### 2.4 提示词模板（`harness/prompt-templates.ts`）

PromptTemplate = `{ name, description, content }`。

支持参数替换：`$1`、`$@`、`$ARGUMENTS`、`${@:N}`、`${@:N:L}`。

通过 `harness.promptFromTemplate(name, args)` 调用，自动展开模板中的参数占位符。

### 2.5 压缩/Compaction（`harness/compaction/`）

当对话上下文接近模型窗口上限时，将历史消息替换为结构化摘要。

**配置：** `DEFAULT_COMPACTION_SETTINGS = { enabled: true, reserveTokens: 16384, keepRecentTokens: 20000 }`

**流程：**
1. `shouldCompact()` — 判断 contextTokens > contextWindow - reserveTokens
2. `prepareCompaction()` — 分析 session tree，找到 cut point：
   - `findCutPoint()` — 从后往前累加 token 估算，找到保留 keepRecentTokens 的切点
   - 切点必须在 valid 切割点（user/assistant/bashExecution/custom 等消息）
   - 如果切点不是 user 消息开头，则 `isSplitTurn`（分割了一个正在进行中的 turn）
3. `compact()` — 调用 LLM 生成总结：
   - 有前次总结时用 UPDATE 格式（保留已有信息 + 新增）
   - 无前次总结时从头生成（Goal / Constraints / Progress / Decisions / Next Steps / Context）
   - `isSplitTurn` 时额外用简短格式总结 turn prefix
4. 结果持久化为 `compaction` entry，附带 `readFiles` 和 `modifiedFiles` 详情
5. 下次 `buildContext()` 时从 compaction entry 之后开始重建消息

**Token 估算：** 字符数 / 4，图片占 4800 token。如果有实际 LLM usage 数据则优先使用。

### 2.6 会话系统（`harness/session/`）

**存储方案：**

| 方案 | 文件 | 用途 |
|------|------|------|
| JSONL（生产） | `jsonl-storage.ts` + `jsonl-repo.ts` | 以 JSONL 文件持久化（每行一个 entry），按 cwd 分目录 |
| Memory（测试） | `memory-storage.ts` + `memory-repo.ts` | 内存实现，用于测试和 harness 抽象 |

**Session Tree 结构：**
```
root
├── leaf → message (user: "hello")
│   ├── leaf → message (assistant: "...")
│   │   ├── leaf → model_change (provider: "anthropic", modelId: "claude")
│   │   ├── leaf → compaction (summary: "...", firstKeptEntryId: "abc")
│   │   ├── leaf → message (user: "do X")
│   │   │   └── leaf → message (assistant: "...")
│   │   └── leaf → branch_summary (fromId: "...", summary: "switched from Y")
│   │       └── leaf → ...
```

树是**仅追加**结构：每个 entry 有 parentId，leaf entry 指向当前活跃路径的末端。分支切换通过 `moveTo(targetEntryId)` 创建新的 leaf 指向目标。

**Session 类（`session.ts`）：**
- `buildContext()` — 从当前 leaf 到根遍历，重建 `{ messages, thinkingLevel, model }`
- `getBranch()` — 获取当前活跃分支的所有 entry
- `moveTo()` — 支持分支导航：跳转到任意 entry 并可选生成 branch summary

**SessionRepo** — 管理 session 文件的生命周期：`create`、`open`、`list`、`delete`、`fork`。

### 2.7 Proxy 代理模式（`proxy.ts`）

`streamProxy()` 替代默认的 `streamSimple`，将 LLM 请求通过 HTTP server 转发：
```
Agent → fetch(PROXY_URL/api/stream) → Server → LLM Provider
```

proxy 事件类型去掉 `partial` 字段（由客户端重建），减小带宽占用。`ProxyAssistantMessageEvent` 定义了完整的 wire protocol。

### 2.8 Harness 完整交互周期

以用户输入 "修改 README" 为例：

```
1. 用户调用 harness.prompt("修改 README")
2. harness 变为 "turn" phase
3. createTurnState: session.buildContext() → 获取 messages + model + thinkingLevel
4. 调用 systemPrompt callback 生成 system prompt（含 skills、工具列表等）
5. executeTurn:
   a. emit "before_agent_start" hook（可修改 prompt/注入消息）
   b. 如果有 nextTurn 队列消息，前置插入
   c. runAgentLoop(messages, context, loopConfig, ...)
      - streamFn 由 harness 创建，注入 before_provider_request / before_provider_payload hooks
      - convertToLlm 转换 AgentMessage → Message
      - transformContext 调用 "context" hook
      - 每轮: LLM streaming → assistant message → tool calls
      - 工具执行期间调用 "tool_call" / "tool_result" hooks
      - turn_end: 持久化消息+模型变更到 session，emit "save_point"
      - prepareNextTurn: createTurnState() 刷新上下文
      - getSteeringMessages: drain() steer queue
      - getFollowUpMessages: drain() follow-up queue
   d. agent_end: flushPendingSessionWrites, phase → idle, emit "settled"
6. 返回最后一个 assistant 消息给调用者
```

**消息持久化策略：** 运行时（phase !== idle）的消息写入暂存在 `pendingSessionWrites` 队列，turn_end 或 agent_end 时批量 flush 到 session storage。

---

## 三、架构总结

```
                    ┌──────────────────────────────────┐
                    │        AgentHarness               │
                    │  session / compaction / hooks     │
                    │  skills / templates / queues      │
                    ├──────────────────────────────────┤
                    │           Agent                   │
                    │  state / prompts / steer/followUp │
                    ├──────────────────────────────────┤
                    │        Agent Loop                 │
                    │  runLoop / stream / executeTools  │
                    ├──────────────────────────────────┤
                    │         pi-ai (streamSimple)      │
                    │    Message[] → LLM Provider       │
                    └──────────────────────────────────┘
```

**设计原则：**
- **事件驱动**：整个循环通过 `AgentEvent` 事件流通信，UI 层 subscribe 即可获得实时状态
- **契约式不抛异常**：LLM 调用、消息转换、工具 hook 都必须返回默认可处理的值而不是抛异常
- **端到端 abort**：单个 AbortController 信号贯穿整个 run，所有异步操作共享
- **可扩展消息**：declaration merging 允许应用增加自定义消息类型而无需修改 agent 源码
- **脚本化 hook**：AgentHarness 暴露 extensive hook 系统，允许外部注入逻辑而不侵入核心循环
- **分支式会话**：仅追加的 tree 结构支持时间旅行式的会话导航和分支回退
