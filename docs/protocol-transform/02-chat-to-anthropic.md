# Anthropic Messages ↔ OpenAI Chat Completions 协议转换详解

> 对应源码：`src-tauri/src/proxy/providers/transform.rs`（1816 行）
> 场景：客户端发出 Anthropic Messages 格式请求（Claude Code 原生协议），上游供应商只支持 OpenAI Chat Completions 格式（OpenRouter 及各类 OpenAI 兼容供应商）

## 总览：为什么需要这层转换

Claude Code 只会说 Anthropic Messages 协议。但相当多的模型供应商（OpenRouter、以及绝大多数第三方 OpenAI 兼容网关）只实现 Chat Completions 协议——这是目前业界最通用的最小公分母协议。这一层转换要解决三个层面的差异：

1. **结构差异**：Anthropic 把 `system` 独立于 `messages`，content 是带类型的 block 数组（text/image/tool_use/tool_result/thinking...）；Chat Completions 把 system 也塞进 messages 数组，content 通常是纯字符串，工具调用是 message 级别的独立字段。
2. **能力差异**：Anthropic 的 `thinking`/`redacted_thinking` block 在 Chat Completions 里没有原生对应物；Anthropic 的 `tool_result` 可以在一条消息里塞多个、可以携带图片，Chat Completions 的 `tool` 角色消息一条只对应一个 tool_call、content 只能是字符串。
3. **兼容性差异**：即使都遵循 Chat Completions 规范，不同网关对 schema 关键字、`content` 是否允许数组形式、`cache_control` 等 Anthropic 专属字段的容忍度也不一样。

设计上这一层转换体现的是"**最小转换、兼容性优先**"：结构性转换尽量保持无损（tool_use/tool_result 的 ID、参数都完整保留），但遇到协议里没有对应字段的内容（主要是 thinking），**默认直接丢弃**——只有识别出特定供应商（DeepSeek/Kimi/Moonshot 等）才会启用非标准的 `reasoning_content` 字段做部分保留。这跟 `03-anthropic-responses-bidirectional.md` 里"用 base64 编码做到完全无损"的思路是两种不同的取舍：那边判断"目标协议虽然没有原生字段，但至少有一个能塞任意字符串的字段（signature/data）可以藏完整数据"，这边的判断是"Chat Completions 的字段都有明确语义，硬塞编码数据风险更高，不如坦然接受有损"。

---

## 一、从 Claude Code 请求看 Anthropic Messages 全貌

> 转换入口函数：`anthropic_to_openai_with_reasoning_content`（line 128-229）。本节格式与 `01` 篇对齐：先放一个带注释的完整请求示例，再逐字段列出处理逻辑和源码行号，最后两句话收束。

#### 总：一个真实请求长什么样

```jsonc
// ================ Claude Code 发出的 POST /v1/messages ================
// 注释标注了每个字段在 Chat Completions 方向映射到的目标及转换逻辑行号。
{
  // ─── 模型标识 ──────────────────────────────
  // → Chat: "model": "claude-sonnet-5"（直接透传，line 135-137）
  "model": "claude-sonnet-5",

  // ─── 系统提示（独立顶层字段）──────────────
  // → Chat: messages[0] = {role:"system", content:"..."}
  // string 直接用作 content；content-block 数组时逐项提取 text。
  // 空值不生成 system 消息。每段 text 先过 strip_leading_anthropic_billing_header
  // 剥除动态轮转的计费元数据行以提高上游 prefix cache 命中率（#2350）。
  // （line 142-159）
  "system": "You are a coding assistant with file access.",

  // ─── 对话历史 ──────────────────────────────
  // → Chat: "messages": [...]（convert_message_to_openai，line 162-168，
  //   normalize_openai_system_messages 合并多条 system 到开头，line 171）
  "messages": [
    // ── user 消息，单字符串 content ──────
    // → Chat: {role:"user", content:"Read README.md"}
    {"role": "user", "content": "Read README.md"},

    // ── assistant 消息，含 tool_use + text ──
    // tool_use → Chat assistant 消息的 tool_calls[] 数组
    // （line 399-411：id/name → id/function.name，
    //  input(JSON对象) → arguments(JSON字符串，canonical_json_string 稳定排序)）
    // text → Chat content_parts[] 或纯字符串（行数 381-385）
    {"role": "assistant", "content": [
      {"type": "text", "text": "Let me read the file."},
      {"type": "tool_use", "id": "call_read", "name": "Read",
       "input": {"file_path": "/path/to/file", "limit": 2000}}
    ]},

    // ── user 消息含 image + tool_result ────
    // image → Chat image_url content part ({type:"image_url", image_url:{url:"data:...;base64,..."}})
    // （line 386-397）
    // tool_result → 独立的 Chat role:"tool" 消息（line 412-428）
    //   - content 为字符串时直接复制
    //   - content 为数组（多模态混合）时 canonical_json_string 序列化成纯文本（有损降级）
    {"role": "user", "content": [
      {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": "iVBORw0..."}},
      {"type": "tool_result", "tool_use_id": "call_read", "content": "File contents here."}
    ]},

    // ── assistant 消息含 thinking + text ────
    // thinking → 默认丢弃（line 430-437：文中收集到 reasoning_parts 但只在
    //   preserve_reasoning_content=true 时才输出到 reasoning_content 扩展字段）。
    //   若整条消息只有 thinking/redacted_thinking 无其他 block，消息完全消失。
    // （详见 §3.5）
    {"role": "assistant", "content": [
      {"type": "thinking", "thinking": "The file contains..."},
      {"type": "text", "text": "The file contains a list of items."}
    ]}
  ],

  // ─── 最大输出 token ────────────────────────
  // → Chat: o-series 模型 → max_completion_tokens，其余 → max_tokens（line 176-181）
  "max_tokens": 32000,

  // ─── Thinking 控制 ─────────────────────────
  // → Chat: resolve_reasoning_effort() 将 Anthropic thinking 映射为
  //   reasoning_effort 字段（line 82-112，仅 supports_reasoning_effort 模型才写入，
  //   详见 §5）
  //   {type:"adaptive"}                  → "xhigh"
  //   {type:"enabled", budget: <4000}   → "low"
  //   {type:"enabled", budget: 4000-15999} → "medium"
  //   {type:"enabled", budget: >=16000} → "high"
  //   output_config.effort 显式值优先于 thinking.type 推断
  "thinking": {"type": "enabled", "budget_tokens": 8192},

  // ─── 直接透传参数 ──────────────────────────
  // 以下字段不做类型映射或值域转换，原样复制到 Chat 请求体（line 183-193）
  "temperature": 0.7,
  "top_p": 0.9,

  // ─── 结构映射参数 ──────────────────────────
  // stop_sequences(复数) → stop(单数)（line 189-191）
  "stop_sequences": ["END"],
  // stream → 直接透传（line 192-193）
  "stream": true,

  // ─── 工具定义 ──────────────────────────────
  // → Chat: tools[{type:"function", function:{name, description, parameters}}]
  // 每个 Anthropic tool 映射为一个 Chat function tool：
  //   name → function.name
  //   description → function.description
  //   input_schema → function.parameters（经 clean_schema 补全 type:object，剥离 format:uri）
  // type=="BatchTool" 的条目整体过滤（Anthropic 专有，Chat 无对应概念）
  // （line 203-222）
  "tools": [
    {"name": "get_weather", "description": "获取天气",
     "input_schema": {"type": "object", "properties": {"city": {"type": "string"}}}},
    {"type": "BatchTool", "name": "batch_read"}  // 这个会被过滤掉
  ],

  // ─── 工具选择策略 ──────────────────────────
  // → Chat: tool_choice（map_tool_choice_to_chat，line 273-294）
  //   "any" / {type:"any"} → "required"
  //   {type:"tool", name:"X"} → {type:"function", function:{name:"X"}}
  //   "auto" / "none" → 同名透传
  "tool_choice": "auto"
}
```

> **关于 `temperature`**：Anthropic `temperature` 和 `top_p` 无条件透传到 Chat（line 183-187）。OpenAI o-series 模型实际不支持 `temperature`，但这个过滤属于路由/供应商能力层职责，不在这个纯结构转换函数里处理。

`content` block type 速查表（`convert_message_to_openai`，line 348-491）：

| block type | Anthropic 关键字段 | Chat 映射 |
|---|---|---|
| `text` | `text`（字符串） | `content` 数组中的一个 text part |
| `image` | `source`: `{type:"base64", media_type, data}` | `{type:"image_url", image_url:{url:"data:...;base64,..."}}` |
| `tool_use` | `id`, `name`, `input`（JSON 对象） | message 级别 `tool_calls[]`：`{id, type:"function", function:{name, arguments(JSON字符串)}}` |
| `tool_result` | `tool_use_id`, `content`（字符串或数组）, `is_error`? | 独立 `role:"tool"` 消息, `content`→字符串(数组则序列化) |
| `thinking` | `thinking`（文本）, `signature`?（签名） | 默认丢弃；`preserve_reasoning_content=true` 时→ `reasoning_content` |
| `redacted_thinking` | `data`（加密内容） | 默认丢弃；同上模式下注入 `"[redacted thinking]"` 占位 |

#### 分：逐字段转换逻辑

`anthropic_to_openai_with_reasoning_content`（line 128-229）中每个字段的处理：

```
字段                    处理                                              源码行
──────────────────────────────────────────────────────────────────────────────────
model                  直接透传                                          135-137
system                 strip_leading_anthropic_billing_header() 后        142-159
                       逐条塞入 Chat messages[] 头部 role:"system"
messages               convert_message_to_openai()                      162-168
                       逐条转换为 Chat messages（block 类型映射见上方速查表）
normalize_system       normalize_openai_system_messages()                 171
                       多个 system 消息合并为一条放到数组开头
max_tokens             o-series 模型 → max_completion_tokens，              176-181
                       其余 → max_tokens
temperature            直接透传                                          183-184
top_p                 直接透传                                          186-187
stop_sequences        → stop（复数→单数）                                189-191
stream                直接透传                                          192-193
thinking              reasoning_effort 映射（仅支持模型才注入）               197-201
                       映射表：adaptive→"xhigh"，budget<4000→"low"，
                       4000-15999→"medium"，≥16000→"high"
tools                  BatchTool 过滤 +                                 203-222
                       input_schema → function.parameters
                       （clean_schema: 补 type:object，去 format:uri）
tool_choice            Anthropic "any"→"required"，                      225-226
                       {type:"tool"}→{type:"function",function:{name}}
──────────────────────────────────────────────────────────────────────────────────
cache_control         所有位置（system/message/tool 定义）
                      都**静默丢弃**（测试 test_regression_gh3805_* 验证）
──────────────────────────────────────────────────────────────────────────────────
其余 Anthropic 字段   元数据类字段（metadata / stop_reason 等）不经过本函数
                      处理，由调用链上层决定
```

#### 总：三条核心差异

Anthropic → Chat 转换的本质就是把三种"Chat 没有"的东西处理掉：

1. **结构重组**：`system` 从独立顶层合并进 `messages[0]`；`tool_result` 从一条 user 消息的 content block 拆成独立 `role:"tool"` 消息；`tool_use` 从 content block 提升到 `message.tool_calls[]`。
2. **向下兼容**：thinking/redacted_thinking 默认丢弃（仅对特定供应商开启 reasoning_content 文本保留）；BatchTool 过滤；cache_control 全部剥离。
3. **输出友好**：单 text block 的 content 序列化为纯字符串而非数组（兼容严格上游）；JSON 对象参数用 `canonical_json_string` 稳定排序（提升上游 prefix cache 命中）。

---

## 二、正常化处理：System 与 Billing Header

### 2.1 System 消息整合：`normalize_openai_system_messages`（line 296-345）

多个 system block 先各自生成一条消息，随后统一合并：0 条不处理；1 条则确保它在数组开头；多条则用 `\n` 拼接所有内容合并为一条并放在开头，原始多条被移除。

```json
// 输入 messages: [{"role":"system","content":"Rule A"}, {"role":"user","content":"Hello"}, {"role":"system","content":"Rule B"}]
// 输出 messages: [{"role":"system","content":"Rule A\nRule B"}, {"role":"user","content":"Hello"}]
```

### 2.2 Billing Header 剥离：`strip_leading_anthropic_billing_header`（line 18-47）

Claude Code 会在 system prompt 开头动态注入一行 `x-anthropic-billing-header:` 计费元数据（内含随请求轮转的 `cch=` 值）。如果原样转发进 OpenAI 的 `messages`/`instructions`，这个不断变化的前缀会让每次请求的 prompt 开头都不同，破坏上游的 prefix cache 命中率（issue #2350）。

规则：**只有**当 system 文本**以** `x-anthropic-billing-header:` 开头时才处理，找到这一行的换行符，把这一整行（含换行符）剥掉，保留后面的正文。非开头位置出现的类似文本不受影响，因为那可能是用户自己写的提示词内容。

---

## 三、Content Block 逐类型映射（`convert_message_to_openai`，line 348-491）

### 3.1 text block

```json
// 输入: {"type":"text","text":"Hello world"}
```
如果这是消息里唯一的 block，输出简化为字符串 `"content":"Hello world"`；如果有多个 block，输出数组形态 `[{"type":"text","text":"Hello world"}]`。**无损**，只是丢弃了 `cache_control` 等附加字段。

单 block 简化为纯字符串的原因：某些严格校验的上游（GLM/Qwen）会拒绝 `content` 为数组格式的请求（对应回归测试 `test_regression_gh3805`）。

### 3.2 image block

```json
// 输入: {"type":"image","source":{"type":"base64","media_type":"image/png","data":"iVBORw0..."}}
// 输出: {"type":"image_url","image_url":{"url":"data:image/png;base64,iVBORw0..."}}
```
Anthropic 用分离字段（`media_type` + `data`），OpenAI 用组合的 data URI。**语义无损**，只是编码格式从分离变成拼接。代码只处理 base64 来源，因为传到代理层的图片基本已经是客户端做过 base64 编码的。

### 3.3 tool_use → tool_calls

```json
// 输入 (assistant message 的 content[] 里):
{"type":"tool_use","id":"call_123","name":"get_weather","input":{"location":"Tokyo"}}

// 输出 (提升到 message 级别的 tool_calls 数组):
"tool_calls": [{"id":"call_123","type":"function","function":{"name":"get_weather","arguments":"{\"location\":\"Tokyo\"}"}}]
```
关键差异：Anthropic 的 `input` 是 JSON 对象，OpenAI 的 `arguments` 是**字符串化**的 JSON（用 `canonical_json_string` 做稳定排序序列化，利于上游 cache 命中）。**语义无损**，只是类型从对象变成字符串。

### 3.4 tool_result → 独立的 tool 角色消息

```json
// 输入 (user message 的 content[] 里):
{"type":"tool_result","tool_use_id":"call_123","content":"Sunny, 25°C"}

// 输出 (独立消息，从原 message 里提升出来):
{"role":"tool","tool_call_id":"call_123","content":"Sunny, 25°C"}
```

**核心语义差异**：Anthropic 允许**同一条** user 消息的 content 数组里塞多个 tool_result；OpenAI 要求每个工具结果都是**独立的** `role:"tool"` 消息。转换时把每个 tool_result 从原消息里拆出来单独成消息（line 424-428）。

**多模态 tool_result 的有损降级**：如果 `content` 是数组（Anthropic 支持图片+文字混合的工具结果），整个数组会被 `canonical_json_string` 序列化成一段 JSON 字符串塞进 OpenAI 的 `content`（字符串字段）：

```json
// 输入: {"type":"tool_result","tool_use_id":"call_456","content":[
//   {"type":"text","text":"Result text"},
//   {"type":"image","source":{"type":"base64","media_type":"image/png","data":"..."}}
// ]}
// 输出: {"role":"tool","tool_call_id":"call_456","content":"[{\"text\":\"Result text\"},{\"source\":{...},\"type\":\"image\"}]"}
```
**这是有损的**：图片从"可被模型看到的多模态内容"降级成"一段包含 base64 字符串的纯文本"，下游模型很可能根本看不出这是图片。原因是 OpenAI 的 `tool` 角色消息的 `content` 协议上只接受字符串，没有多模态数组的位置。

### 3.5 thinking / redacted_thinking —— 本模块信息损耗最大的部分

**默认路径（`preserve_reasoning_content=false`）**：
- `thinking` block 的文本被收集但**不输出**到最终消息
- `redacted_thinking` block **直接忽略**
- 如果一条消息**只有** thinking/redacted_thinking block，没有 text/tool_use/tool_result，这条消息在转换后**完全消失**（不产出任何 Chat 消息）——对应测试 `test_anthropic_to_openai_skips_thinking_only_message`

**`preserve_reasoning_content=true` 路径**（仅当识别为 Moonshot/Kimi/DeepSeek 等供应商时启用）：
- `thinking` 文本 → 拼接进 `reasoning_content` 字段
- `redacted_thinking` → 替换成占位符文本 `"[redacted thinking]"`
- 仅在 `role=="assistant"` 且有 `tool_calls` 时才输出 `reasoning_content`（如果没有 reasoning 文本但有 tool_calls，注入占位符 `"tool call"`，因为这类供应商的 thinking 模型会拒绝没有 reasoning_content 的工具调用消息）

**结论**：默认情况下 Claude 的 extended thinking 内容在转 Chat Completions 时**完全丢失**；即使开启保留模式，也只保留纯文本，结构信息（签名、block 类型标记）同样丢失。这是本文档系列里*唯一*一处"完全没有编码兜底方案、纯粹接受信息丢失"的转换点，与 `05-reasoning-thinking-tools-cross-protocol.md` 里 Anthropic↔Responses 方向靠 base64 编码做到无损形成鲜明对比——因为 Responses 协议有 `encrypted_content` 这种"天然就是不透明数据"的字段可以借用，Chat Completions 没有。

---

## 四、JSON Schema 清洗：`clean_schema` / `clean_schema_inner`（transform.rs:493-525）

### 4.1 根 schema 补齐（仅 `is_root=true` 时）

```json
// 输入: {"properties":{"location":{"type":"string"}}}   // 缺 type
// 输出: {"type":"object","properties":{"location":{"type":"string"}}}
```
Anthropic 的 `input_schema` 允许省略根 `type`（隐式当作 object），但 OpenAI 的 `function.parameters` 要求必须显式声明 `type:"object"`，缺失时自动补齐；`properties` 缺失同样补齐为 `{}`。

### 4.2 移除 `format: "uri"`

```json
// 输入: {"type":"string","format":"uri"}
// 输出: {"type":"string"}
```
只精确删除值为 `"uri"` 的 `format` 字段（其余 format 值不受影响），因为部分上游对函数参数 schema 中的 `format:"uri"` 会报错拒绝。

### 4.3 递归但不越权

对嵌套的 `properties.*` 和 `items` 递归应用同样规则，但递归调用时 `is_root=false`，所以不会给嵌套 schema 强行补 `type`。

**不做清理的字段**：`$schema`、`additionalProperties`、`anyOf`/`oneOf`/`allOf`、`enum`/`const`/`default`/`description` 均**原样保留**——这是"最小干预"策略的体现，只处理已知会导致上游拒绝的两个具体问题，其余原样透传，不做过度设计。

调用点：`transform.rs:214`（本文件 Anthropic→OpenAI 方向）以及 `transform_responses.rs`（Anthropic→Responses 方向，见另一篇文档）复用同一个函数。

---

## 五、Reasoning Effort 处理

### 5.1 模型识别：`is_openai_o_series` / `supports_reasoning_effort`

```
is_openai_o_series(model):
  model.len() > 1 && model.starts_with('o') && 第二个字符是数字
  "o1" → true, "o1-preview" → true, "o3-mini" → true
  "gpt-4o" → false (以 'g' 开头), "o" → false (长度不够)

supports_reasoning_effort(model):
  is_openai_o_series(model)
  || model 以 "gpt-" 开头 且 紧跟的数字字符 >= '5'
  "gpt-5" → true, "gpt-5.1" → true, "gpt-4" → false, "gpt-4o" → false
```

### 5.2 effort 数值来源：`resolve_reasoning_effort`（transform.rs:82-112）

**优先级 1 — 显式 `output_config.effort`**：
```
"low"→"low", "medium"→"medium", "high"→"high", "max"→"xhigh"（唯一发生改名的一档）
其他值 → 不注入
```

**优先级 2 — 从 `thinking.type` + `budget_tokens` 推断**：
```
thinking.type == "adaptive"                    → "xhigh"
thinking.type == "enabled", budget_tokens<4000  → "low"
thinking.type == "enabled", 4000<=budget<16000  → "medium"
thinking.type == "enabled", budget_tokens>=16000 → "high"
thinking.type == "enabled", 无 budget_tokens     → "high"
thinking.type == "disabled" 或无 thinking 字段    → 不注入
```

**语义错配提示**：Anthropic 的 `budget_tokens` 控制的是"思考过程本身可以占用多少 token 篇幅"（一个可观测的输出量上限），OpenAI 的 `reasoning_effort` 控制的是"模型内部推理链条的强度"（不可观测的内部预算）。两者不是同一个维度的概念，这里的区间映射是**启发式**近似，不保证语义精确对应。

注入门槛：只有 `supports_reasoning_effort(model)` 为真时才会写入 `reasoning_effort` 字段（transform.rs:196-201），非推理模型即使配了 thinking 也不会收到这个字段。

---

## 六、tool_choice 映射：`map_tool_choice_to_chat`（transform.rs:273-294）

| Anthropic | OpenAI Chat |
|---|---|
| `"auto"` / `{"type":"auto"}` | `"auto"` |
| `"any"` / `{"type":"any"}` | `"required"` |
| `"none"` / `{"type":"none"}` | `"none"` |
| `{"type":"tool","name":"X"}` | `{"type":"function","function":{"name":"X"}}` |

注意两点：OpenAI 没有区分"必须调用工具但可以是任意一个"（Anthropic 的 `any`）和"必须调用工具"（`required`）这两种语义，映射时近似合并；以及 Anthropic 的 tool_choice 是扁平结构，OpenAI Chat 要求嵌套一层 `function`（这一点与 Responses API 的扁平结构不同，参见另一篇文档）。

---

## 七、响应反方向转换：`openai_to_anthropic`（transform.rs:528-723）

### 7.1 reasoning_content → thinking block

```json
// 输入: {"choices":[{"message":{"reasoning_content":"Need to check...","content":"Hello"}}]}
// 输出 content 数组: [{"type":"thinking","thinking":"Need to check..."}, {"type":"text","text":"Hello"}]
```
只要 `reasoning_content` 非空，就在 content 数组最前面插入一个**不带 signature** 的 thinking block。

### 7.2 message.content → text blocks

字符串 content 直接包成一个 text block；数组 content 逐项映射（`text`/`output_text` → text block，`refusal` → 当作 text block 输出）。message 级别的 `refusal` 字段同样转成 text block。

### 7.3 tool_calls → tool_use blocks

```json
// 输入: {"tool_calls":[{"id":"call_123","function":{"name":"get_weather","arguments":"{\"location\":\"Tokyo\"}"}}]}
// 输出: [{"type":"tool_use","id":"call_123","name":"get_weather","input":{"location":"Tokyo"}}]
```
关键反向操作：`arguments`（JSON 字符串）解析回 `input`（JSON 对象）；如果 `arguments` 不是合法 JSON，降级为空对象 `{}`（不中断转换）。旧版 `function_call` 字段（非 `tool_calls`）同样兼容处理。

### 7.4 finish_reason → stop_reason

```
"stop"           → "end_turn"
"length"         → "max_tokens"
"tool_calls"     → "tool_use"
"function_call"  → "tool_use"
"content_filter" → "end_turn"
未知值           → "end_turn"（附加告警日志）
finish_reason 为 null 但检测到 tool_use block → 推断为 "tool_use"
```

### 7.5 usage 字段映射——三桶模型的核心难点

OpenAI 的 `prompt_tokens` 是"包含缓存"的总量，Anthropic 的 `input_tokens` 是"刨除缓存"的净量，需要做减法还原：

```
cache_read     ← usage.cache_read_input_tokens 或 usage.prompt_tokens_details.cached_tokens
cache_creation ← usage.cache_creation_input_tokens 或 prompt_tokens_details.cache_write_tokens 或 input_tokens_details.cache_write_tokens
input_tokens (Anthropic) = prompt_tokens.saturating_sub(cache_read).saturating_sub(cache_creation)
output_tokens (Anthropic) = completion_tokens
```
用 `saturating_sub` 而非普通减法，是为了在上游 cache token 统计出现异常（比如 cache_read + cache_creation 之和超过 prompt_tokens）时防止下溢——对应测试 `test_openai_to_anthropic_clamps_input_when_cache_exceeds_prompt` 验证了极端情况下 `input_tokens` 会被安全地钳制到 0，而不是 panic 或产生巨大的负数溢出值。

`stop_sequence` 字段在非流式响应里始终设为 `null`（流式路径在专门的 streaming.rs 里处理，见 `04-sse-streaming-transform.md`）。

---

## 八、其他边界情况

### 8.1 BatchTool 过滤（transform.rs:206-207）

`tools[]` 中 `type=="BatchTool"` 的条目在转换时被整条过滤掉——这是 Anthropic 专有的批量工具调用机制，OpenAI 协议里没有对应概念。

### 8.2 cache_control 全面丢弃

无论出现在 system block、message content block 还是 tools 定义上，`cache_control` 字段在转换过程中都被静默忽略（不会出现在任何输出里）。这不是遗漏，是主动设计：对应的回归测试 `test_regression_gh3805_no_cache_control_leak_to_openai`（line 1327）专门验证了转换后各处都不含这个字段，测试命名指向 issue #3805，结合上下文判断可能是为了兼容严格的上游供应商对未知字段的拒绝行为。

### 8.3 温度类参数无条件透传

`temperature`/`top_p` 原样透传，不做模型能力校验（比如 OpenAI o-series 实际不支持 `temperature`，但这个过滤属于路由/供应商能力层的职责，不在这个纯结构转换函数里处理）。

### 8.4 model 字段留白处理

如果 Anthropic 请求体里没有 `model` 字段，输出结果里也不设置该字段——因为模型名映射统一交给上游 `proxy::model_mapper` 模块处理，这一层转换函数专注结构转换，不掺和模型路由逻辑。

---

## 总结：设计哲学回顾

1. **最小转换原则**：只改结构，不碰模型名映射（那是 `model_mapper` 的职责），也不做模型能力校验（那是路由层的职责）。
2. **兼容性优先于表达力**：单 text block 简化为字符串而非数组、多模态 tool_result 序列化降级为纯文本、cache_control 全面丢弃——所有这些选择都是"让请求更可能被严格上游接受"而不是"尽可能保留原始结构"。
3. **thinking 的有损是明知且默认的**：默认丢弃，仅对特定供应商开启部分文本保留，这跟本系列另一篇文档里 Anthropic↔Responses 方向的无损编码策略形成对照——**是否值得做无损桥接，取决于目标协议是否天然存在"能装任意数据的不透明字段"**。Chat Completions 没有，所以这里选择坦然接受损耗而不是勉强编码。
4. **缓存亲和性是隐藏的一条主线**：`strip_leading_anthropic_billing_header` 和 `canonical_json_string` 的稳定排序，两处独立的设计都是为了让转换后的请求在上游获得更好的 prefix cache 命中率，这类"看不见的性能优化"在协议转换代码里占了不小的篇幅。
