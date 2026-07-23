# SSE 流式响应跨协议转换详解

> 对应源码：`src-tauri/src/proxy/providers/streaming.rs`（1248 行）+ `streaming_responses.rs`（2197 行）+ `streaming_codex_anthropic.rs`（1194 行）+ `streaming_codex_chat.rs`（1256 行）+ `codex_responses_sse.rs`（约 400 行，共享事件构造器）
> 场景：客户端发起流式请求（`stream: true`）时，代理需要边收边转，把上游协议格式的 SSE 事件流实时转换成客户端协议格式的 SSE 事件流

## 总览：流式转换比非流式转换难在哪

前三篇文档讲的转换函数都有一个共同前提：**完整的请求体/响应体已经在手上**，可以随意读取任意字段、任意重排数组顺序。流式转换没有这个前提——上游的响应是一个事件序列，一次只到达一小片内容，而且到达顺序和分片方式完全由上游决定，代理必须在"信息不完整"的情况下持续做出正确的转换决策。

这就带来三个非流式转换不需要面对的新问题：

1. **状态机代替一次性转换**：不能等所有内容都到了再转换，必须维护一个"当前正在输出第几个 block、这个 block 是什么类型、已经发过 start 事件没有"的状态机，每来一个上游事件就驱动状态机往前走一步。
2. **索引重新编号的实时性**：上游的工具调用数组下标和目标协议的 content block 下标计数体系不同（比如 Anthropic 的 index 是文本块和工具块混合计数，Chat 的 tool_calls index 是独立计数），必须在事件到达的当下就分配好目标协议的编号，不能等看完全部内容再统一编号。
3. **分片到达的顺序错位**：工具调用参数通常是先来一个不完整的 id/name，再陆续来若干个 JSON 片段；reasoning 的摘要文本是逐字流式到达，但摘要背后的加密内容/签名往往只在这个 block 结束时一次性给出——转换器要正确处理这种"部分内容早到、部分内容晚到"的不对称性。

四个转换方向分别对应四对协议组合：

| 文件 | 方向 |
|---|---|
| `streaming.rs` | Chat Completions SSE → Anthropic Messages SSE |
| `streaming_responses.rs` | OpenAI Responses SSE → Anthropic Messages SSE |
| `streaming_codex_anthropic.rs` | Anthropic Messages SSE → OpenAI Responses SSE（前者的镜像方向） |
| `streaming_codex_chat.rs` | Chat Completions SSE → OpenAI Responses SSE |

（Gemini 方向的 `streaming_gemini.rs` 不在本文重点覆盖范围。）

---

## 一、三种协议的 SSE 事件类型全景

### 1.1 Anthropic Messages 流式协议

采用 `event:` + `data:` 两行一组的命名事件格式：

| 事件名 | 触发时机 | 关键字段 |
|---|---|---|
| `message_start` | 流开始 | `message.id`, `message.model`, `message.role="assistant"`, `message.usage`(初始 output_tokens=0) |
| `content_block_start` | 每个 block 开始 | `index`, `content_block{type}` |
| `content_block_delta` | 增量内容 | `index`, `delta{type}` |
| `content_block_stop` | block 结束 | `index` |
| `message_delta` | 最后一个 block 停止后的全局状态变更 | `delta.stop_reason`, `usage`(最终统计) |
| `message_stop` | 流结束 | 无数据 |
| `error` | 错误 | `error.type`, `error.message` |

`delta.type` 的四种子类型：`text_delta`(携带 `text`)、`input_json_delta`(携带 `partial_json`，可以是不完整的 JSON 片段)、`thinking_delta`(携带 `thinking`)、`signature_delta`(携带 `signature`，通常一次性给出而非分片)。

`content_block.type` 的四种：`text`、`tool_use`(`id`/`name`/`input`)、`thinking`(`thinking`/`signature`)、`redacted_thinking`(`data`)。

**完整示例**：
```
event: message_start
data: {"type":"message_start","message":{"id":"msg_1","model":"claude","usage":{"input_tokens":12,"output_tokens":0}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":3}}

event: message_stop
data: {"type":"message_stop"}
```

### 1.2 OpenAI Chat Completions 流式协议

没有 `event:` 行，只有 `data:` 行，结尾是 `data: [DONE]`。单个 chunk：

```json
{
  "id": "chatcmpl_...", "model": "gpt-4o",
  "choices": [{
    "delta": {
      "content": "Hello",
      "reasoning": "...",                    // 或别名 reasoning_content
      "tool_calls": [{
        "index": 0,
        "id": "call_...", "type": "function",       // 只在首次出现
        "function": {"name": "tool_name", "arguments": "{\"loc"}  // 增量 JSON 片段
      }]
    },
    "finish_reason": "stop"                  // 或 tool_calls/function_call/length/content_filter，最后一个 chunk 才有
  }],
  "usage": {"prompt_tokens":100,"completion_tokens":50,"prompt_tokens_details":{"cached_tokens":60,"cache_write_tokens":30}}
}
```

要点：`delta.tool_calls` 是数组，每项有独立 `index`（并行工具调用场景下有多个）；第一次出现某个 index 时可能同时带 `id`/`type`/`function.name`，后续只带 `function.arguments` 的增量片段；`finish_reason` 在某些供应商（OpenRouter）可能重复出现，需要去重处理。

### 1.3 OpenAI Responses 流式协议

命名事件格式，完整事件清单（按功能分组）：

**生命周期类**（不直接产生 Anthropic 输出）：`response.created`、`response.in_progress`、`response.completed`、`response.incomplete`、`response.failed`、`error`

**输出构建类**：

| 事件名 | 映射到 Anthropic |
|---|---|
| `response.output_item.added` | `content_block_start` |
| `response.output_text.delta` | `content_block_delta`(text_delta) |
| `response.output_text.done` | `content_block_stop` |
| `response.output_item.done` | `content_block_stop`(工具调用/reasoning 收尾) |
| `response.function_call_arguments.delta` | `content_block_delta`(input_json_delta) |
| `response.function_call_arguments.done` | `content_block_stop` |
| `response.reasoning_summary_text.delta`（及别名 `response.reasoning_text.delta`/`response.reasoning.delta`） | `content_block_delta`(thinking_delta) |
| `response.reasoning_summary_text.done`（及别名） | 若无 delta 则补发 thinking_delta 后关闭 |

三个 reasoning delta 事件名是**兼容别名**关系，代码统一处理，这是因为不同实现（官方 API vs 各类第三方网关）对同一个语义用了不同的事件名。

**完整示例**：
```
event: response.created
data: {"type":"response.created","response":{"id":"resp_1","model":"gpt-4o"}}

event: response.output_item.added
data: {"type":"response.output_item.added","item":{"id":"fc_1","type":"function_call","call_id":"call_1","name":"get_weather"}}

event: response.function_call_arguments.delta
data: {"type":"response.function_call_arguments.delta","item_id":"fc_1","delta":"{\"city\":\"Tokyo\"}"}

event: response.function_call_arguments.done
data: {"type":"response.function_call_arguments.done","item_id":"fc_1"}

event: response.completed
data: {"type":"response.completed","response":{"status":"completed","usage":{"input_tokens":12,"output_tokens":3}}}
```

---

## 二、四个转换器的状态机设计

### 2.1 `streaming.rs`：Chat → Anthropic

核心状态字段（局部变量，无独立结构体）：

| 字段 | 作用 |
|---|---|
| `next_content_index: u32` | 全局递增的 Anthropic content block 编号计数器 |
| `has_sent_message_start` / `has_emitted_message_delta` / `has_sent_message_stop` | 三处去重标记，防止上游重复发终态事件导致协议违规 |
| `pending_message_delta` | 缓存 (stop_reason, usage)，延迟到 `[DONE]` 才真正发出 |
| `current_non_tool_block_type` / `current_non_tool_block_index` | 追踪当前是文本块还是思考块，用于判断要不要先关闭再开新的 |
| `tool_blocks_by_index: HashMap<usize, ToolBlockState>` | key 是 Chat 的 tool_calls 下标，value 是该工具块的完整状态 |

`ToolBlockState` 结构体（streaming.rs:88-99）：

```rust
struct ToolBlockState {
    anthropic_index: u32,       // 分配到的 Anthropic content block 编号
    id: String, name: String,   // 从分片 delta 里逐步累积
    started: bool,               // 是否已发出 content_block_start
    pending_args: String,        // start 之前先到达的参数片段，暂存
    consecutive_whitespace: usize, aborted: bool,  // 见 §6.5 Copilot 空白 bug 防护
}
```

**为什么要延迟发送 `message_delta`**：代码注释明确写了原因——某些上游（比如 OpenRouter 的 kimi-k2.6）会在 tool_use 之后发送**多个**带 `finish_reason` 的 chunk，但 Anthropic 协议规定一条消息流只能有一个 `message_delta`，重复发送会导致 Claude Code 直接 abort 连接。所以只处理第一个 `finish_reason`（用 `has_emitted_message_delta` 去重），并且把 `message_delta` 缓存到收到 `[DONE]` 时才真正发出——这样能确保携带的 usage 统计是最终完整值，而不是中途某个不完整的快照。

### 2.2 `streaming_responses.rs`：Responses → Anthropic

同样是局部变量驱动的状态机，关键字段：

| 字段 | 作用 |
|---|---|
| `index_by_key: HashMap<String, u32>` | 通用的"任意 key → Anthropic index"映射表 |
| `tool_index_by_item_id` / `tool_name_by_index` / `tool_args_by_index` | 三张表分别按 item_id 查 index、按 index 查工具名、按 index 累积参数字符串 |
| `reasoning_index_by_item_id` / `reasoning_item_by_index` / `reasoning_text_by_index` | reasoning 相关的三张对应表 |
| `has_substantive_output: bool` | 是否已经产生过任何实质性输出（用于判断流异常终止时该报 incomplete 还是 failed） |

**`resolve_content_index` 的编号分配逻辑**：优先按 `content_part_key` 查表复用已分配的编号；查不到则看有没有 `fallback_open_index` 可以复用；都没有才从 `next_content_index` 分配一个新编号。这套"key 优先、fallback 兜底、最后才新分配"的三级查找顺序，是为了兼容"部分上游给了明确 key、部分上游啥都不给"的现实。

### 2.3 `streaming_codex_anthropic.rs`：Anthropic → Responses

这是唯一使用**独立结构体**（而非一堆局部变量）的实现，因为它的状态需要在多个处理函数之间传递：

```rust
struct AnthropicToResponsesState {
    response_started: bool, completed: bool,
    response_id: String, model: String,
    next_output_index: u32,
    blocks: BTreeMap<u64, BlockState>,      // key = Anthropic content block index
    output_items: Vec<(u32, Value)>,         // 已完成的 output item，按 output_index 排序
    anthropic_usage: Map<String, Value>,
    stop_reason: Option<String>,
    tool_context: CodexToolContext,          // 复用 01 篇文档提到的同一个工具上下文类型
}
```

`BlockState` 每个字段的用途：
```rust
struct BlockState {
    kind: BlockKind,              // Text / Tool / Thinking 三态枚举
    output_index: u32, item_id: String, call_id: String, name: String,
    accum: String,                 // 累积的文本/参数/thinking 内容（用于 close 时的完整值兜底）
    start_input: String,           // content_block_start 里携带的完整 input（无 delta 时的 fallback）
    source_block: Value,           // 保留原始 content_block 副本，供重建 reasoning item 用
    has_visible_summary: bool, done: bool,
}
```

用 `BTreeMap` 而不是 `HashMap` 存 blocks 是因为最终 finalize 时需要按 index 顺序处理，`BTreeMap` 天然维护了排序，省去了额外的 sort 步骤。

### 2.4 `streaming_codex_chat.rs`：Chat → Responses

```rust
struct ChatToResponsesState {
    response_started: bool, completed: bool,
    next_output_index: u32,
    text: TextItemState, reasoning: ReasoningItemState,
    inline_think: InlineThinkState,           // 见 §5.4，识别内联 <think> 标签的独立子状态机
    tools: BTreeMap<usize, ToolCallState>,     // key = Chat tool_calls 下标
    next_tool_index_to_add: usize,             // 见 §4.4，强制按顺序释放工具调用
    tool_context: CodexToolContext,
}
```

---

## 三、工具调用参数的分片拼接策略：直接转发，不等完整

**核心结论：四个转换方向全部采用"边到边转发"策略——增量片段到达就立即包装转发，不等 JSON 拼完整。**

这是因为目标协议（Anthropic 的 `input_json_delta.partial_json`、Responses 的 `function_call_arguments.delta`）本身设计上就允许接收不完整的 JSON 片段，不要求每次 delta 都是合法 JSON。代码里确实也在累积参数字符串（比如 `tool_args_by_index`/`BlockState.accum`），但累积的**目的不是等完整了再发**，而是：

1. 供 block 结束时做 `canonicalize`/`sanitize`（比如 Read 工具的参数需要清理空 pages 字段）
2. 供"上游跳过了 delta 事件、只在 done 事件里给完整参数"这种网关兼容场景做 fallback
3. 供工具调用尚未确定 id/name 时暂存参数，等 id/name 到齐后一次性把暂存内容连同后续新到的内容一起发出

**Read 工具和 custom tool 是唯二的例外**：这两类工具的参数不走实时转发，而是在 close_block 时才统一处理并一次性发出完整参数——因为 Read 工具的参数需要清理规范化，custom tool 的参数需要重新包装格式，这些操作在"只有半个 JSON 片段"时做不了。（这是对代码分支行为的归纳总结，源码里没有直接的注释点明"为什么只有这两类需要特殊处理"，但从各自的清理/包装逻辑能推断出上述原因。）

---

## 四、索引对齐：四种协议各自的编号体系怎么互相映射

### 4.1 Chat → Anthropic

Chat 的 `tool_calls[i].index` 是独立于文本的并行调用序号；Anthropic 的 content block index 是文本块和工具块**混合计数**的。转换器维护 `tool_blocks_by_index: HashMap<Chat_index, ToolBlockState>`，每次遇到新的 Chat index 就从全局的 `next_content_index` 分配一个新的 Anthropic 编号并递增计数器。

```
时序: [文本delta] [tool_call index=0] [tool_call index=1]
Anthropic 编号分配: 0(text) → 1(tool, 对应 chat index 0) → 2(tool, 对应 chat index 1)
```

**Late start 场景**：如果参数片段先到，但 id/name 还没凑齐（这在某些供应商的分片方式下会发生），参数暂存在 `pending_args` 里；一旦 id/name 到齐，直接用预先分配好的 `anthropic_index` 发 `content_block_start`，再把暂存的内容连带发出。

### 4.2 Responses → Anthropic

用 `item_id` 查表找到预分配的 index，逻辑和上面类似，只是 key 从"数组下标"换成了"字符串 item_id"。

### 4.3 Anthropic → Responses

这个方向相对简单：Anthropic 已经明确给出了 `data.index`，转换器只需要为每个 block 分配一个新的 Responses `output_index`（从 `next_output_index` 递增获取），是"一对一重新编号"而不需要复杂的查找逻辑。

### 4.4 Chat → Responses

`streaming_codex_chat.rs` 这里有一个独特的约束——**工具必须按 Chat index 严格顺序添加**，即使 tool index 1 的 id/name 比 tool index 0 先到达，也必须等 tool 0 就绪之后才能给 tool 1 分配 output_index。`next_tool_index_to_add` 这个字段就是用来强制这个顺序的守门变量。这是因为 Responses 协议对 output_index 的语义要求"先到先编号"，如果乱序分配会导致下游按 index 重建顺序时出错。

---

## 五、Reasoning/Thinking 在流式过程中的处理

### 5.1 Chat → Anthropic：直接转发，无编解码

```
delta.reasoning = "推理增量文本"
  → content_block_start(type="thinking", thinking="")   （如果前一个 block 不是 thinking 类型，先关旧的再开新的）
  → content_block_delta(type="thinking_delta", thinking="推理增量文本")
```
没有涉及签名/加密内容，纯文本直接转发。这跟非流式的 `02-chat-to-anthropic.md` 里"thinking 默认丢弃"的策略不同——流式场景下，如果上游 Chat 供应商真的通过 `delta.reasoning` 发了推理内容，是会被转发保留的（只是仍然没有 signature）。

### 5.2 Responses → Anthropic：摘要流式到达，加密内容一次性到达的不对称处理

摘要文本走 `response.reasoning_summary_text.delta`（及别名）逐字到达，直接转发成 `thinking_delta`。但 `encrypted_content` 这个字段**不会分片流式给出**，只会在 `response.output_item.done` 事件里一次性给出完整值。转换器在收到 done 事件、发现 reasoning item 带 `encrypted_content` 时：

1. 调用 `encode_openai_reasoning_item(item)` 把**整个** reasoning item（含刚刚累积的 summary 文本和这次才拿到的 encrypted_content）序列化 base64
2. 如果 block 已经开着（因为之前发过 delta）：发 `signature_delta`，把编码结果塞进 signature 字段
3. 如果 block 还没开（没有 summary 文本，一开始就直接给了 encrypted_content）：发 `content_block_start(type="redacted_thinking", data=编码结果)`

这一步用的正是 `03-anthropic-responses-bidirectional.md` 里讲过的通道 1（`reasoning_bridge.rs`）编码机制——流式场景和非流式场景共用同一套编解码函数，只是流式场景下"什么时候调用它"由 done 事件触发，而不是像非流式那样在整个响应体已经完整时统一处理。

### 5.3 Anthropic → Responses：signature_delta 不产生输出事件，只更新内部状态

```
content_block_delta(type="thinking_delta") → 立即转发成 response.reasoning_summary_text.delta
content_block_delta(type="signature_delta") → 不产生任何输出事件，只更新 BlockState.source_block["signature"]
content_block_stop → 触发 close_block，此时才调用 responses_reasoning_item_from_anthropic_block(&source_block)
                     用累积的完整 source_block（含 thinking 文本 + signature）一次性构造完整的 Responses reasoning item
                     （这里走的是 03 篇文档提到的通道 2 编码，encrypted_content = "ccswitch-anthropic-thinking-v1:" + base64）
```

这个设计的核心考量：`signature_delta` 通常是整个 thinking block 结束前的最后一条 delta，携带一个不可分割的完整签名值。转换器选择"先攒着，close 时才一次性构造输出 item"而不是"收到就转发"，是因为构造完整的 Responses reasoning item（需要同时具备 summary 文本和 encrypted_content）必须等两部分数据都到齐。

### 5.4 InlineThink 机制：识别混在普通文本里的 `<think>` 标签

**问题背景**：部分供应商（如 MiniMax）通过标准 Chat Completions 接口暴露推理能力，但不用独立的 `reasoning_content` 字段，而是直接把思考过程用 `<think>...</think>` 包裹后混进 `delta.content` 文本流。转换器必须实时识别这种内联标记并把它拆分成"推理部分"和"正文部分"分别转发。

**识别规则**（`leading_think_prefix_decision`）：
```
buffer 去掉前导空白后:
  为空                        → NeedMore（继续等更多字符）
  以 "<think>" 开头            → Reasoning（确认是思考标记）
  是 "<think>" 的前缀子串       → NeedMore（比如目前只到 "<thi"，还不能判断，继续等）
  其他                         → Text（确认不是思考标记，走普通文本路径）
```

**状态转换时序**：
```
delta="<think>\nNeed"          → buffer累积，判定=NeedMore（不完整，暂不发任何事件）
delta=" context.</think>\n\npong" → buffer变为完整，drain_complete_inline_think 成功拆出:
                                    reasoning="Need context." → 发 reasoning_summary_text_delta
                                    answer="pong"             → 发 output_text.delta
```

这套机制本质是一个嵌套在主状态机里的小型子状态机（`InlineThinkMode` 枚举 + `InlineThinkState`），处理的是"同一个字符流里混杂着两种语义内容，需要按标签边界实时拆分"这个通用问题。当工具调用等其他事件打断这个检测过程时，`flush_inline_think_at_boundary` 会强制把缓冲区里已有的内容按当前判定结果发出去，不会无限期等待。

---

## 六、其他关键设计点

### 6.1 消息生命周期事件的映射时机

| 上游事件 | 目标事件 | 元数据来源 |
|---|---|---|
| Chat: 第一个有内容的 chunk | `message_start` | `chunk.id`→`message.id`, `chunk.model`→`message.model` |
| Responses: `response.created` | `message_start` | `response.id`→`message.id` |
| Anthropic: `message_start` | `response.created` + `response.in_progress` | `message.id`→`response.id`(加 `resp_` 前缀) |

**惰性发送策略**：`message_start`/`response.created` 不是收到上游第一个事件就立即发出，而是等到第一个**实质性输出事件**（delta 类）到达时才补发（`ensure_response_started`/`has_sent_message_start` 检查）。这样可以确保发出的元数据（model、id）确实是从上游拿到的，也避免了"上游只发了纯元数据、没有任何实际输出内容"的边缘情况下产生一个看起来成功但完全空洞的消息生命周期。

### 6.2 usage 统计的延迟传递

三个协议的 usage 到达时机都不固定（可能中途某个 chunk 带了、也可能只有最后一个带），四个转换器统一采用"边到达边更新缓存变量、真正发送延迟到确认是最终值的那一刻"的策略。比如 `streaming.rs` 的 `latest_usage`会随着任何带 usage 的 chunk 更新，但只在 `[DONE]` 到达时才把它塞进最终的 `message_delta` 事件。

### 6.3 stop_reason / finish_reason 映射（流式场景专用表，与非流式场景基本一致）

**Chat → Anthropic**：`tool_calls`/`function_call`→`tool_use`，`stop`→`end_turn`，`length`→`max_tokens`，`content_filter`→`end_turn`。

其余方向的映射表与对应的非流式转换（`02-chat-to-anthropic.md` §7.4、`03-anthropic-responses-bidirectional.md` §7.1-7.2）基本一致，流式场景只是把判断时机从"响应体完整时一次性判断"变成"收到终态事件时判断"。

### 6.4 错误处理：统一走"构造目标协议 error/failed 事件 + 标记终止"两步

四个转换器在遇到上游流本身出错（`stream.next()` 返回 `Err`）或上游发来错误事件（`error`/`response.failed`）时，统一执行两个动作：构造目标协议对应的错误终态事件（Anthropic 是 `event: error`，Responses 是 `response.failed`），以及设置一个终止标记（`stream_ended_with_error`/`terminated`/`completed`）防止后续再误发一个"成功"的终态事件把错误状态覆盖掉。

流被截断但没有明确错误信号的场景（连接中断、没等到终态事件流就断了）区分处理：已经产生过实质输出（`has_substantive_output`）→ 尝试发 incomplete 结束；完全没有输出 → 发 failed。

### 6.5 Copilot 无限空白 bug 防护

`streaming.rs` 里的 `ToolBlockState.consecutive_whitespace` 专门用来对付一个观察到的真实供应商 bug：某些 Copilot 代理场景下，工具调用参数流会开始持续输出空白字符而不停止。转换器统计连续空白字符数，一旦超过阈值（`INFINITE_WHITESPACE_THRESHOLD`），就把这个工具块标记为 `aborted`，跳过它后续的所有 delta，避免响应流无限膨胀导致客户端卡死或内存暴涨。这是一处纯粹为了应对已知供应商异常行为而写的防御代码，不是协议规范要求的。

---

## 总结：流式转换的设计哲学

1. **边到边转发是默认策略，缓冲是例外**：绝大多数 delta 内容（文本、reasoning 摘要、工具参数 JSON 片段）到达即转发，最大程度降低端到端延迟。只有当目标协议要求"必须一次性给出完整值"（reasoning 的 encrypted_content、Read 工具的规范化参数）时才攒起来延迟发送。
2. **状态机是处理"部分信息、乱序到达"的唯一手段**：四个转换器不约而同地引入了状态结构体（或等价的局部变量集合），本质都是在回答同一个问题——"当前这个上游事件，应该对应目标协议里正在进行的哪一个 block/item，这个 block/item 现在处于什么阶段"。
3. **去重和终态保护贯穿所有实现**：无论方向，"确保 message_start/response.created 只发一次"、"确保终态事件只发一次"都是硬性要求，因为目标协议的客户端（Claude Code / Codex CLI）对协议违规（重复终态事件）的容忍度很低，轻则显示异常、重则直接断开连接。
4. **兼容性别名和 fallback 逻辑是现实所迫**：三个 reasoning delta 事件别名、`done` 事件里可能补发完整参数、Copilot 空白 bug 防护，这些都不是协议规范的一部分，而是实际接入了大量不完全遵循规范的上游供应商之后，用测试和真实流量喂出来的防御代码。
