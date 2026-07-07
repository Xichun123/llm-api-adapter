# SSE Wire Format Reference

Detailed byte-level format requirements for Server-Sent Events across all three LLM API protocols.

## Table of Contents
1. [General SSE Rules](#general-sse-rules)
2. [OpenAI Chat Completions SSE](#openai-chat-completions-sse)
3. [Anthropic Messages SSE](#anthropic-messages-sse)
4. [OpenAI Responses API SSE](#openai-responses-api-sse)
5. [Keepalive and Timeout](#keepalive-and-timeout)
6. [Common Pitfalls](#common-pitfalls)

---

## General SSE Rules

SSE is a W3C standard. Each event is:
```
[event: <type>\n]
data: <payload>\n
\n
```

Key rules:
- Lines end with `\n` (LF), not `\r\n`
- Each event terminates with a blank line (`\n\n`)
- `event:` line is optional (OpenAI Chat omits it; Anthropic and Responses use it)
- `data:` can span multiple lines: each line starts with `data: `
- A comment line starts with `:` — used for keepalive (`: keepalive\n\n`)

### Flushing

After writing each SSE event to the response writer, you **must** flush the buffer. Without flushing, events accumulate in the server/framework's output buffer and the client sees nothing until the buffer fills or the stream ends.

In Go: `flusher.Flush()` via `http.Flusher` interface
In Python (FastAPI/Starlette): `yield` in `StreamingResponse` auto-flushes
In Node.js (Express): `res.flush()` or `res.flushHeaders()` + `res.write()`
In Rust (axum): use `axum::response::Sse` which auto-flushes

### Headers

```http
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no
```

`X-Accel-Buffering: no` is critical when behind nginx — without it, nginx buffers the entire response before forwarding.

---

## OpenAI Chat Completions SSE

Format: `data: <JSON>\n\n` (no `event:` field)

### Wire example (raw bytes)

```
data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","created":1700000000,"model":"gpt-5-mini","choices":[{"index":0,"delta":{"role":"assistant"},"logprobs":null,"finish_reason":null}]}\n
\n
data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","created":1700000000,"model":"gpt-5-mini","choices":[{"index":0,"delta":{"content":"Hi"},"logprobs":null,"finish_reason":null}]}\n
\n
data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","created":1700000000,"model":"gpt-5-mini","choices":[{"index":0,"delta":{"content":" there!"},"logprobs":null,"finish_reason":null}]}\n
\n
data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","created":1700000000,"model":"gpt-5-mini","choices":[{"index":0,"delta":{},"logprobs":null,"finish_reason":"stop"}],"usage":{"prompt_tokens":8,"completion_tokens":3,"total_tokens":11}}\n
\n
data: [DONE]\n
\n
```

### Chunk fields

| Field | First chunk | Middle chunks | Final chunk |
|-------|-------------|---------------|-------------|
| `id` | Present | Same value | Same value |
| `object` | `"chat.completion.chunk"` | Same | Same |
| `created` | Unix timestamp | Same | Same |
| `model` | Model ID | Same | Same |
| `system_fingerprint` | Optional | Same | Same |
| `choices[0].index` | `0` | `0` | `0` |
| `choices[0].delta.role` | `"assistant"` | Absent | Absent |
| `choices[0].delta.content` | Absent | Text fragment | Absent |
| `choices[0].delta.tool_calls` | Absent | Tool call fragments | Absent |
| `choices[0].logprobs` | `null` | `null` | `null` |
| `choices[0].finish_reason` | `null` | `null` | `"stop"` / `"tool_calls"` / `"length"` |
| `usage` | Absent | Absent | Token counts (optional) |

### Tool call streaming detail

Tool calls arrive incrementally across multiple chunks:

```
# Chunk 1: tool call header (id, name, empty arguments)
delta.tool_calls: [{"index":0,"id":"call_abc","type":"function","function":{"name":"get_weather","arguments":""}}]

# Chunk 2-N: argument fragments
delta.tool_calls: [{"index":0,"function":{"arguments":"{\"loc"}}]
delta.tool_calls: [{"index":0,"function":{"arguments":"ation\":"}}]
delta.tool_calls: [{"index":0,"function":{"arguments":" \"SF\"}"}}]

# Final chunk
delta: {}, finish_reason: "tool_calls"
```

Accumulate `arguments` fragments per tool call `index`. The `id` and `name` only appear once.

---

## Anthropic Messages SSE

Format: `event: <type>\ndata: <JSON>\n\n`

### Complete event sequence

```
event: message_start\n
data: {"type":"message_start","message":{...}}\n
\n
event: ping\n
data: {"type":"ping"}\n
\n
event: content_block_start\n
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}\n
\n
event: content_block_delta\n
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}\n
\n
event: content_block_stop\n
data: {"type":"content_block_stop","index":0}\n
\n
event: message_delta\n
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":15}}\n
\n
event: message_stop\n
data: {"type":"message_stop"}\n
\n
```

### Event type reference

| Event | Payload | When |
|-------|---------|------|
| `message_start` | Full message object (empty content, usage with input_tokens) | First event |
| `ping` | `{"type":"ping"}` | After message_start |
| `content_block_start` | Block type + index | Before each content block |
| `content_block_delta` | Delta for current block | During content generation |
| `content_block_stop` | Block index | After each content block |
| `message_delta` | stop_reason + output usage | After all blocks |
| `message_stop` | `{"type":"message_stop"}` | Final event |

### Content block types and their deltas

| Block type | Start payload | Delta type | Delta payload |
|------------|--------------|------------|---------------|
| `text` | `{"type":"text","text":""}` | `text_delta` | `{"type":"text_delta","text":"..."}` |
| `thinking` | `{"type":"thinking","thinking":"","signature":""}` | `thinking_delta` | `{"type":"thinking_delta","thinking":"..."}` |
| `thinking` (close) | — | `signature_delta` | `{"type":"signature_delta","signature":""}` |
| `tool_use` | `{"type":"tool_use","id":"...","name":"...","input":{}}` | `input_json_delta` | `{"type":"input_json_delta","partial_json":"..."}` |

### Block ordering rules

1. Thinking blocks come first (if thinking enabled)
2. Text blocks come after thinking
3. Tool use blocks come after text
4. Each block increments the index
5. Close reasoning block (`signature_delta` + `content_block_stop`) before starting text block

---

## OpenAI Responses API SSE

Format: `event: <type>\ndata: <JSON>\n\n`

This is the strictest format. **Every event must include `sequence_number`.**

### Complete event sequence for text response

```
event: response.created\n
data: {"type":"response.created","sequence_number":0,"response":{...full response object...}}\n
\n
event: response.in_progress\n
data: {"type":"response.in_progress","sequence_number":1,"response":{...same...}}\n
\n
event: response.output_item.added\n
data: {"type":"response.output_item.added","sequence_number":2,"output_index":0,"item":{"type":"message","id":"msg_abc","status":"in_progress","role":"assistant","content":[]}}\n
\n
event: response.content_part.added\n
data: {"type":"response.content_part.added","sequence_number":3,"item_id":"msg_abc","output_index":0,"content_index":0,"part":{"type":"output_text","text":"","annotations":[]}}\n
\n
event: response.output_text.delta\n
data: {"type":"response.output_text.delta","sequence_number":4,"item_id":"msg_abc","output_index":0,"content_index":0,"delta":"Hello"}\n
\n
event: response.output_text.done\n
data: {"type":"response.output_text.done","sequence_number":N,"item_id":"msg_abc","output_index":0,"content_index":0,"text":"Hello there!"}\n
\n
event: response.content_part.done\n
data: {"type":"response.content_part.done","sequence_number":N+1,"item_id":"msg_abc","output_index":0,"content_index":0,"part":{"type":"output_text","text":"Hello there!","annotations":[]}}\n
\n
event: response.output_item.done\n
data: {"type":"response.output_item.done","sequence_number":N+2,"output_index":0,"item":{...completed item...}}\n
\n
event: response.completed\n
data: {"type":"response.completed","sequence_number":N+3,"response":{...completed response...}}\n
\n
```

### Response object — required fields

The `response` object in `response.created` and `response.completed` must include all these fields (use `null` for absent values):

```json
{
  "id": "resp_abc",
  "object": "response",
  "created_at": 1700000000,
  "status": "in_progress",
  "model": "claude-sonnet-4-6",
  "output": [],
  "usage": null,
  "error": null,
  "incomplete_details": null,
  "instructions": null,
  "metadata": {},
  "parallel_tool_calls": true,
  "temperature": 1.0,
  "tool_choice": "auto",
  "tools": [],
  "top_p": 1.0,
  "max_output_tokens": null,
  "previous_response_id": null,
  "reasoning": {"effort": null, "summary": null},
  "store": true,
  "truncation": "disabled",
  "user": null
}
```

### Event field matrix

| Event | `sequence_number` | `item_id` | `output_index` | `content_index` |
|-------|-------------------|-----------|----------------|-----------------|
| `response.created` | Yes | No | No | No |
| `response.in_progress` | Yes | No | No | No |
| `response.output_item.added` | Yes | No | Yes | No |
| `response.content_part.added` | Yes | Yes | Yes | Yes |
| `response.output_text.delta` | Yes | Yes | Yes | Yes |
| `response.output_text.done` | Yes | Yes | Yes | Yes |
| `response.content_part.done` | Yes | Yes | Yes | Yes |
| `response.output_item.done` | Yes | No | Yes | No |
| `response.function_call_arguments.delta` | Yes | Yes | Yes | No |
| `response.function_call_arguments.done` | Yes | Yes | Yes | No |
| `response.completed` | Yes | No | No | No |

---

## Keepalive and Timeout

For long-running requests (e.g., complex reasoning), the stream may go silent for 30+ seconds. Without keepalive, reverse proxies (nginx, Cloudflare) may close the connection.

### SSE comment keepalive

Send a comment every 5-15 seconds during silent periods:
```
: keepalive\n
\n
```

This is a valid SSE comment (starts with `:`) — clients ignore it, but it keeps the TCP connection alive.

### Implementation pattern

```pseudocode
start keepalive timer (every 5s):
    write ": keepalive\n\n"
    flush

on each real SSE event:
    reset keepalive timer

on stream end:
    stop keepalive timer
```

### Client disconnect handling

When the client disconnects mid-stream:
- Detect via write error or context cancellation
- Stop the keepalive timer
- Close the upstream connection (don't waste upstream tokens)
- Clean up resources

---

## Common Pitfalls

### 1. Missing double newline
Each SSE event ends with `\n\n`. A single `\n` means the event is not yet complete — the client will buffer and wait for more data.

### 2. No flush after write
Writing to the response writer doesn't mean the data reaches the client. You must flush after each event.

### 3. Writing headers after first write
HTTP headers must be written before the first body byte. If you call `http.Error()` after streaming has started, it becomes part of the body and corrupts the SSE stream.

### 4. Buffer size too small
Scanner/reader buffers must be large enough for the largest possible SSE line. A single chunk with a large tool call argument can exceed 64KB. Use at least 1MB buffer.

### 5. JSON in SSE data field
The JSON must be on a single line (no pretty-printing). Newlines in JSON would be interpreted as new SSE fields.

### 6. Forgetting `data: [DONE]` (OpenAI Chat)
Some clients (like OpenAI's own SDK) wait for `[DONE]` to finalize the response. Without it, the client may hang or timeout.

### 7. Inconsistent `id` across chunks (OpenAI Chat)
All chunks in one response must share the same `id`. Generating a new ID per chunk breaks clients that group chunks by ID.

### 8. Missing `event:` field (Anthropic/Responses)
Unlike OpenAI Chat (which only uses `data:`), Anthropic and Responses formats require the `event:` field. Without it, clients can't dispatch events by type.
