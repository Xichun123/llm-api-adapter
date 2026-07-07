---
name: llm-api-adapter
description: >-
  LLM API 适配代理层：OpenAI Chat Completions / Anthropic Messages / OpenAI Responses 三协议适配、SSE 流式传输、账号池重试。适用场景：构建 API 代理、实现 OpenAI/Anthropic 兼容接口、协议格式转换、SSE 流式调试、账号池轮转、Cherry Studio/Cursor/Claude Code 等客户端空响应排查。即使用户只说"做成 OpenAI 兼容"或"加 Anthropic 流式"或"代理在 Cherry Studio 里返回空"，都应使用此技能。
---

# LLM API 适配器 — 协议规格与实现指南

用于构建代理/适配层的实战参考，将任意 LLM 后端暴露为**三种业界标准 API 接口**。语言无关——Go、Python、TypeScript、Rust 或任何 HTTP 服务器均适用。

三种接口：
1. **OpenAI Chat Completions** (`POST /v1/chat/completions`) — 最广泛支持的格式
2. **Anthropic Messages** (`POST /v1/messages`) — Claude 原生 API 格式
3. **OpenAI Responses** (`POST /v1/responses`) — OpenAI 新格式，SSE 要求最严格

通用架构：接收客户端请求 → 转译为上游格式 → 调用上游 → 转译响应 → 返回客户端。魔鬼藏在 SSE 流式细节中。

---

## 架构总览

```
客户端请求（三种格式任一）
  → 根据路径路由到处理器
  → 验证认证（Bearer Token）
  → 将请求转译为上游格式
  → 从账号池选择凭证（LRU 或最高余额优先）
  → 发送到上游 LLM 后端
  → 出错时：分类 → 重试下一账号 或 返回客户端
  → 成功时：将响应转译回客户端格式
  → 流式发送 SSE 事件（或非流式返回 JSON）
  → 客户端接收响应
```

### 需实现的端点

| 路径 | 方法 | 格式 | 认证头 |
|------|------|------|--------|
| `/v1/models` | GET | OpenAI | `Authorization: Bearer <key>` |
| `/v1/chat/completions` | POST | OpenAI Chat | `Authorization: Bearer <key>` |
| `/v1/messages` | POST | Anthropic Messages | `Authorization: Bearer <key>` 或 `x-api-key: <key>` |
| `/v1/responses` | POST | OpenAI Responses | `Authorization: Bearer <key>` |

### SSE 流式必需 HTTP 头

每个流式端点在写入 body 前**必须**设置：

```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

若位于 Nginx 后方，还需设置 `X-Accel-Buffering: no` 以防缓冲。

### CORS

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization, X-API-Key
```

`OPTIONS` 预检请求返回 200，无 body。

---

## 1. OpenAI Chat Completions (`/v1/chat/completions`)

最简单的接口——若上游已是 OpenAI 兼容格式，基本直接透传。

### 请求格式

```json
{
  "model": "<MODEL_NAME>",
  "messages": [
    {"role": "system", "content": "你是有帮助的助手。"},
    {"role": "user", "content": "你好"},
    {"role": "assistant", "content": "你好！"},
    {"role": "user", "content": "最近怎么样？"}
  ],
  "stream": true,
  "max_tokens": 4096,
  "temperature": 0.7,
  "tools": [...],
  "tool_choice": "auto"
}
```

### 流式 SSE 格式

每行格式为 `data: <JSON>\n\n`，以 `data: [DONE]\n\n` 结束。

```
data: {"id":"chatcmpl-abc123","object":"chat.completion.chunk","created":1700000000,"model":"<MODEL_NAME>","choices":[{"index":0,"delta":{"role":"assistant"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc123","object":"chat.completion.chunk","created":1700000000,"model":"<MODEL_NAME>","choices":[{"index":0,"delta":{"content":"你好"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc123","object":"chat.completion.chunk","created":1700000000,"model":"<MODEL_NAME>","choices":[{"index":0,"delta":{},"finish_reason":"stop"}],"usage":{"prompt_tokens":10,"completion_tokens":5,"total_tokens":15}}

data: [DONE]
```

关键字段：
- `id`：同一响应所有 chunk 保持一致（格式：`chatcmpl-<random>`）
- `object`：始终为 `"chat.completion.chunk"`
- `choices[0].delta.role`：仅出现在第一个 chunk（`"assistant"`）
- `choices[0].finish_reason`：最终 chunk 前为 `null`，之后为 `"stop"` / `"tool_calls"` / `"length"`
- `usage`：仅在最终 chunk 中（部分上游可能省略）

### 流式 Tool Calls

```
data: {"choices":[{"delta":{"tool_calls":[{"index":0,"id":"call_abc","type":"function","function":{"name":"get_weather","arguments":""}}]}}]}

data: {"choices":[{"delta":{"tool_calls":[{"index":0,"function":{"arguments":"{\"loc"}}]}}]}

data: {"choices":[{"delta":{"tool_calls":[{"index":0,"function":{"arguments":"ation\": \"SF\"}"}}]}}]}
```

`id` 和 `name` 仅在该 tool call index 的第一个 chunk 出现，后续 chunk 仅包含 `arguments` 片段，需客户端侧累积拼接。

---

## 2. Anthropic Messages (`/v1/messages`)

### 请求格式

```json
{
  "model": "<MODEL_NAME>",
  "system": "你是有帮助的助手。",
  "messages": [
    {"role": "user", "content": "你好"},
    {"role": "assistant", "content": [{"type": "text", "text": "你好！"}]},
    {"role": "user", "content": [{"type": "text", "text": "最近怎么样？"}]}
  ],
  "max_tokens": 4096,
  "stream": true,
  "thinking": {"type": "enabled", "budget_tokens": 10000},
  "tools": [{"name": "get_weather", "description": "...", "input_schema": {...}}],
  "tool_choice": {"type": "auto"}
}
```

与 OpenAI 的关键差异：
- `system` 是顶层字段（字符串或 `{type:"text", text:"..."}` 块数组）
- `content` 可为字符串或类型化块数组（`text`、`image`、`tool_use`、`tool_result`、`thinking`）
- `tools[].input_schema` 而非 `tools[].function.parameters`
- `tool_choice` 为 `{type: "auto"|"any"|"tool"|"none"}` 而非字符串
- `thinking` 字段用于扩展思考模式

### 流式 SSE 格式 — 完整事件序列

Anthropic SSE 使用 `event: <type>\ndata: <JSON>\n\n` 格式，事件顺序严格：

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_abc","type":"message","role":"assistant","content":[],"model":"<MODEL_NAME>","stop_reason":null,"usage":{"input_tokens":0,"output_tokens":0,"cache_creation_input_tokens":0,"cache_read_input_tokens":0}}}

event: ping
data: {"type":"ping"}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"你好"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":15}}

event: message_stop
data: {"type":"message_stop"}
```

关键细节：
- `message_start` 后**必须**发送 `ping` 事件——Claude Code 等客户端依赖此事件
- 每个内容块：`content_block_start` → 多个 `content_block_delta` → `content_block_stop`
- Block `index` 随每个内容块递增（text、thinking、tool_use）
- `message_delta` 携带最终 `stop_reason` 和输出用量
- `message_stop` 为终止符（**无** `[DONE]` 标记）

### 思考块（扩展思考）

启用思考模式时，`thinking` 块在 text 块之前：

```
event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"thinking","thinking":"","signature":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"thinking_delta","thinking":"让我思考一下..."}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"signature_delta","signature":""}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}
```

关闭 thinking 块前**必须**发送 `signature_delta`，即使签名为空。

### 流式 Tool Use

```
event: content_block_start
data: {"type":"content_block_start","index":1,"content_block":{"type":"tool_use","id":"toolu_abc","name":"get_weather","input":{}}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"{\"location\":"}}

event: content_block_stop
data: {"type":"content_block_stop","index":1}
```

Tool use 作为 stop_reason 时：`message_delta` 中 `stop_reason` = `"tool_use"`。

### stop_reason 映射表（OpenAI ↔ Anthropic）

| OpenAI `finish_reason` | Anthropic `stop_reason` |
|------------------------|------------------------|
| `"stop"` | `"end_turn"` |
| `"tool_calls"` | `"tool_use"` |
| `"length"` | `"max_tokens"` |

---

## 3. OpenAI Responses API (`/v1/responses`)

**最严格的 SSE 格式**。Cherry Studio 等客户端会校验每个字段——缺少 `sequence_number` 或 `item_id` 会导致空白渲染。

### 请求格式

```json
{
  "model": "<MODEL_NAME>",
  "input": "你好",
  "stream": true,
  "max_output_tokens": 4096,
  "instructions": "你是有帮助的助手。",
  "temperature": 0.7
}
```

`input` 可为字符串（简单提示 → 转换为单条 user 消息）或消息对象数组。

### 流式 SSE — 完整事件序列

每个事件格式：`event: <type>\ndata: <JSON>\n\n`。每个 data 负载**必须**包含 `sequence_number`（从 0 递增）。

```
event: response.created
data: {"type":"response.created","sequence_number":0,"response":{"id":"resp_abc","object":"response","status":"in_progress","model":"<MODEL_NAME>","output":[],"usage":null,...}}

event: response.in_progress
data: {"type":"response.in_progress","sequence_number":1,"response":{...}}

event: response.output_item.added
data: {"type":"response.output_item.added","sequence_number":2,"output_index":0,"item":{"type":"message","id":"msg_abc","status":"in_progress","role":"assistant","content":[]}}

event: response.content_part.added
data: {"type":"response.content_part.added","sequence_number":3,"item_id":"msg_abc","output_index":0,"content_index":0,"part":{"type":"output_text","text":"","annotations":[]}}

event: response.output_text.delta
data: {"type":"response.output_text.delta","sequence_number":4,"item_id":"msg_abc","output_index":0,"content_index":0,"delta":"你好"}

event: response.output_text.done
data: {"type":"response.output_text.done","sequence_number":6,"item_id":"msg_abc","output_index":0,"content_index":0,"text":"你好世界！"}

event: response.content_part.done
data: {"type":"response.content_part.done","sequence_number":7,...,"part":{"type":"output_text","text":"你好世界！","annotations":[]}}

event: response.output_item.done
data: {"type":"response.output_item.done","sequence_number":8,...}

event: response.completed
data: {"type":"response.completed","sequence_number":9,"response":{...,"status":"completed","usage":{"input_tokens":10,"output_tokens":5,"total_tokens":15}}}
```

### 客户端强制校验的必需字段

缺少以下字段会导致严格客户端**空响应**：

| 字段 | 位置 | 原因 |
|------|------|------|
| `sequence_number` | 每个 SSE 事件 | 客户端用于排序和去重 |
| `item_id` | `content_part.added`、`output_text.delta/done`、`content_part.done` | 关联内容与父消息 |
| `annotations` | `output_text` 的 `part` 对象中 | 数组（可为空 `[]`）——缺失则解析失败 |
| `response.in_progress` 事件 | `response.created` 之后 | 部分客户端等待此事件才处理内容 |
| 完整的 response metadata | `response.created` 和 `response.completed` 中 | `metadata`、`tools`、`reasoning`、`temperature` 等 |

---

## 格式转换规则

上游与客户端使用不同格式时，需要双向转换。核心要点：

- **Anthropic → OpenAI**：`system` 顶层字段 → `messages[0]` system 消息；`tools[].input_schema` → `tools[].function.parameters`；`tool_choice` 对象 → 字符串/对象；`tool_use`/`tool_result` 块 → `tool_calls` 数组和 `role:"tool"` 消息
- **OpenAI → Anthropic**：`content` 字符串 → `[{type:"text", text:...}]`；`prompt_tokens` → `input_tokens`
- **Responses ↔ Chat**：`input` 字符串 → `messages`；`instructions` → system 消息；`max_output_tokens` → `max_tokens`

> 完整转换映射表和代码模式详见 `references/conversion-patterns.md`。

---

## 错误处理与账号池重试

代理多账号凭证时，需三级错误分类来决定"返回客户端" vs "重试下一账号" vs "禁用账号并重试"。

### 三级错误分类

```
上游返回非 200
  │
  ├─ 第一级：请求过大（403 + "estimated cost"）
  │  → 立即返回错误给客户端
  │  → 任何账号都无法处理，不是账号问题
  │
  ├─ 第二级：额度不足（403 + "insufficient tokens"|"upgrade your plan"|"limit reached"）
  │  → 用下一个账号重试（不禁用当前账号）
  │  → 当前账号可能恢复；其他账号可能有足够额度
  │
  ├─ 第三级：账号耗尽（429 或 402）
  │  → 禁用账号，用下一个重试
  │  → 账号配额已确定用完
  │
  ├─ 认证无效（401）
  │  → 禁用账号，用下一个重试
  │  → Token 过期或被撤销
  │
  └─ 其他错误
     → 返回错误给客户端
     → 未知问题，不要掩盖
```

### 重试循环伪代码

```python
MAX_RETRIES = 10
tried_ids = []

for attempt in range(MAX_RETRIES):
    account = get_active_account(exclude=tried_ids)
    if not account:
        return 503, "无可用活跃账号"
    
    tried_ids.append(account.id)
    mark_account_used(account.id)
    response = call_upstream(account.api_key, request_payload)
    
    if response.status == 200:
        stream_or_return(response)
        return
    
    error_body = response.body
    if is_request_too_large(response.status, error_body):
        return error_to_client(response.status, error_body)   # 不重试
    if is_insufficient_tokens(response.status, error_body):
        continue   # 下一账号，不禁用
    if is_token_exhausted(response.status, error_body):
        disable_account(account.id)
        continue   # 下一账号
    if response.status == 401:
        disable_account(account.id)
        continue
    return error_to_client(response.status, error_body)       # 未知错误

return 503, "所有账号已耗尽"
```

此模式需在所有 6 条代码路径（3 接口 × 流式/非流式）中**一致应用**。

---

## Anthropic 兼容性边缘场景

以下是实际会导致 400 错误或流式中断的问题。

### 孤立 tool_use 块

客户端截断对话历史时，`assistant` 消息可能包含 `tool_use` 块但下一条 `user` 消息中没有对应 `tool_result`。Anthropic API 会拒绝此请求。需扫描消息并插入合成的 `tool_result`：

```json
{
  "type": "tool_result",
  "tool_use_id": "<孤立的_id>",
  "content": "[工具结果不可用 - 对话历史已被截断]",
  "is_error": true
}
```

### cache_control 剥离

若上游不支持 Anthropic 提示缓存，需从 `system` 块、`messages[].content` 块和 `tools` 数组项中移除 `cache_control` 字段。

### thinking 字段剥离

若上游不支持扩展思考，转发前删除请求中的 `thinking` 字段。

### 图片格式转换

- Anthropic: `{type:"image", source:{type:"base64", media_type:"image/png", data:"..."}}`
- OpenAI: `{type:"image_url", image_url:{url:"data:image/png;base64,..."}}`

跨协议转译时需互转。

---

## 模型名称映射

支持短别名和带提供商前缀的名称。映射旧名到当前名：

```
# 别名 → 提供商前缀
claude-sonnet-4-6       → anthropic/claude-sonnet-4-6
gpt-5-mini              → openai/gpt-5-mini

# 旧名 → 当前名
claude-3-5-sonnet-20241022 → anthropic/claude-sonnet-4-5

# 按模式自动前缀
claude-*  → anthropic/
gpt-*     → openai/
gemini-*  → google/
grok-*    → xai/
```

`/v1/models` 端点应返回上游模型和本地别名（去重），使用 TTL 缓存（如 5 分钟）。

---

## 实现检查清单

构建新适配器或审计现有适配器时使用。

### 每端点检查

- [ ] **认证验证** — 处理前检查 Bearer Token
- [ ] **请求解析** — 处理 `stream: true` 和 `stream: false`
- [ ] **模型映射** — 将别名转译为上游模型 ID
- [ ] **格式转译** — 将请求转换为上游格式
- [ ] **错误分类** — 实现全部三级 + 401 处理
- [ ] **重试循环** — 最多 N 个账号，追踪已尝试 ID
- [ ] **SSE 头** — Content-Type、Cache-Control、Connection、X-Accel-Buffering
- [ ] **每个事件后 Flush** — 对实时流式至关重要
- [ ] **用量追踪** — 从上游捕获 token 计数，包含在最终事件中

### 每格式检查

**OpenAI Chat Completions：**
- [ ] 流式透传（若上游 OpenAI 兼容）
- [ ] `data: [DONE]` 终止符
- [ ] 所有 chunk 的 `id` 一致

**Anthropic Messages：**
- [ ] `message_start` → `ping` → 内容块 → `message_delta` → `message_stop`
- [ ] Block index 追踪（每块递增）
- [ ] 思考块关闭前发送 `signature_delta`
- [ ] Tool use: `content_block_start`（类型 `tool_use`）→ `input_json_delta` 块
- [ ] usage 中包含 `cache_creation_input_tokens` 和 `cache_read_input_tokens`（即使为 0）

**OpenAI Responses：**
- [ ] **每个**事件都有 `sequence_number`（从 0 递增）
- [ ] content_part 和 text delta/done 事件都有 `item_id`
- [ ] `response.created` 之后发送 `response.in_progress` 事件
- [ ] 所有 `output_text` part 中包含 `annotations: []`
- [ ] `response.created` 和 `response.completed` 中包含完整 response metadata
- [ ] `function_call` 作为独立 output item（不嵌入 message）

---

## 调试空响应

客户端显示空输出但 curl 正常时，按以下顺序排查：

1. **缺少 `sequence_number`**（Responses API）— Cherry Studio 最常见原因
2. **delta 事件缺少 `item_id`**（Responses API）— 客户端无法关联文本与 item
3. **缺少 `response.in_progress` 事件** — 部分客户端等待此事件
4. **缺少 `annotations: []`**（output_text part 中）— 导致解析失败
5. **缺少 `ping` 事件**（Anthropic）— Claude Code 可能卡住
6. **每个 SSE 事件后未 Flush** — 数据滞留缓冲区
7. **Nginx 缓冲** — 缺少 `X-Accel-Buffering: no` 头
8. **`Content-Type` 错误** — 必须为 `text/event-stream`，而非 `application/json`
9. **缺少双换行** — SSE 事件必须以 `\n\n` 结尾，而非 `\n`

> 更多 SSE 线格式要求详见 `references/sse-wire-format.md`。
> 格式转换代码模式详见 `references/conversion-patterns.md`。
