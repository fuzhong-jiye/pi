# 008 - Tool Search 与 Prompt Caching

> 本篇是关于 **Anthropic API / Claude Code harness** 的机制，非 pi 源码；用于理解"上千工具如何不撑爆上下文、不破坏缓存"。已对照官方文档核实。

## 一、Prompt Caching：前缀缓存

来源：[Prompt caching](https://platform.claude.com/docs/en/docs/build-with-claude/prompt-caching)

- **前缀哈希**：在标了 `cache_control` 的断点处，对**到该块为止的整段前缀**做哈希；后续请求重算前缀哈希比对，命中则复用。最多回看 20 个位置。
- **严格层级顺序**：`Tools → System → Messages`。**前面一层变，后面全失效。**
- **最多 4 个缓存断点**；默认 TTL 5 分钟（复用时免费刷新），可选 1 小时（2× 写入价）。

### 各类改动的失效范围

| 改动 | 影响 |
|------|------|
| **Tool definitions** | **整个缓存失效**（tools+system+messages）——最致命 |
| web search / citations 开关、fast mode | system + messages |
| tool_choice | 仅 messages |
| images 增删 | 仅 messages |
| thinking 参数 | 仅 messages |

**结论**：tool 定义排在最前，任何字节变动会让整条缓存作废。这就是不能把上千工具明文堆进 `tools` 的根本原因。

---

## 二、Tool Search：让"新增 tool"不破坏前缀

来源：[Tool search tool](https://platform.claude.com/docs/en/docs/agents-and-tools/tool-use/tool-search-tool)

### 解决的问题
- **上下文膨胀**：多 MCP server（GitHub/Slack/Sentry…）光定义就可能吃掉 ~55k token。
- **选择精度下降**：工具超过 30-50 个后，模型选对工具的能力明显变差。

### 机制
1. `tools` 里放一个常驻搜索工具：`tool_search_tool_regex_20251119`（Python 正则）或 `tool_search_tool_bm25_20251119`（自然语言）。
2. 其余工具标 `defer_loading: true`。**它们仍写在 `tools` 参数里，但 API 不把它们序列化进用于缓存哈希的 prefix。**
3. 初始模型只看到搜索工具 + 非 deferred 的 3-5 个高频工具。
4. 需要时调搜索工具 → API 返回 3-5 个 `tool_reference` 块 → **就地 append 到对话末尾并展开成完整定义**。
5. 模型从发现的工具中选择并调用。

### 官方原文（关键）
> "Deferred tools are **not included in the system-prompt prefix**. When the model discovers a deferred tool through tool search, the API **appends a `tool_reference` block inline in the conversation**, then expands it into the full tool definition before passing it to Claude. **The prefix is untouched, so prompt caching is preserved.**"

### 为什么"新增 tool 不影响前缀匹配"
- deferred 工具的定义**自始至终不属于被哈希的 prefix**，所以**增删 deferred 工具，prefix 字节零改动** → 缓存照常命中。
- 要用时才以 `tool_reference` 追加到 **messages 尾部**（append-only），永远碰不到前缀。
- 一句话：**所有"新增 tool"的信息都被设计成只出现在缓存断点之后的追加区域。**

### 约束 / 取舍
- 至少要有一个非 deferred 工具（搜索工具自己**绝不能** `defer_loading`）。
- 每个可被搜到的工具必须在 `tools` 里有完整定义（否则 `tool_reference` 报"missing tool definition"）。
- 代价：用 deferred 工具前要先花一次搜索往返；换取前缀缓存长期命中 + 上下文不被撑爆。
- 上限：catalog 最多 10000 工具；每次返回 3-5 个；正则 ≤200 字符。建议 10+ 工具 / 定义 >10k token 时才用。

---

## 三、什么是 `<system-reminder>`

- **来源**：harness（Claude Code 运行外壳）注入到消息流里的标记文本，**不是用户、也不是模型写的**。包在 `<system-reminder>...</system-reminder>` 里，塞进某个 user 回合的消息内容中。
- **用途**：投递动态的、随时变化的上下文/状态。例：可搜的 deferred 工具清单、任务追踪提醒、文件被外部修改、待办状态、当前日期、被召回的 memory 等。
- **语义**：当作**背景信息**，不是用户的直接命令；反映"写入那一刻"的状态，可能过时，引用具体文件/字段要先核实。
- **为什么用它而非 system prompt**：system prompt 在缓存层级靠前，频繁改动会失效缓存；system-reminder 走 **messages 尾部**，append-only，改它不动 prefix——和 deferred 工具"把易变信息挪到尾部"是**同一个缓存友好思路**。

> Claude Code 里看到的那条 "The following deferred tools are now available via ToolSearch … CronCreate, WebFetch …" 就是 harness 在 API 的 `defer_loading` 之上，把可搜工具清单呈现给模型的一层包装；底层落地仍是 `defer_loading` + `tool_reference`。

---

## 一句话总览

> 缓存按前缀失效、tool 定义最致命 → 所以把海量工具标 `defer_loading` 排除出前缀，用搜索工具按需把 `tool_reference` 追加到对话尾部展开；同理，易变状态用 `<system-reminder>` 走尾部下发。**核心手法都是：稳定的放前缀、易变的放尾部。**
