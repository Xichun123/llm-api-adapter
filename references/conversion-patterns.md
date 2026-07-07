# Format Conversion Patterns

Language-agnostic pseudocode for converting between all three LLM API formats.
Real-world patterns extracted from a production proxy handling 470+ accounts.

## Table of Contents
1. [Anthropic Request → OpenAI Request](#anthropic-to-openai-request)
2. [OpenAI Response → Anthropic Response](#openai-to-anthropic-response)
3. [OpenAI Stream → Anthropic Stream](#openai-stream-to-anthropic-stream)
4. [Responses Request → Chat Completions Request](#responses-to-chat-completions-request)
5. [Chat Completions Stream → Responses Stream](#chat-completions-stream-to-responses-stream)
6. [Chat Completions Response → Responses Response](#chat-completions-to-responses-response)
7. [Tool Format Conversion](#tool-format-conversion)

---

## Anthropic to OpenAI Request

```pseudocode
function translateAnthropicToOpenAI(anthropicPayload):
    openai = {
        model: anthropicPayload.model,
        messages: [],
        max_tokens: anthropicPayload.max_tokens ?? 4096,
        stream: false
    }
    
    // Pass through optional params
    if anthropicPayload.temperature: openai.temperature = anthropicPayload.temperature
    if anthropicPayload.top_p: openai.top_p = anthropicPayload.top_p
    
    // System message
    if anthropicPayload.system:
        if typeof system == string:
            openai.messages.push({role: "system", content: system})
        if typeof system == array:
            text = system.filter(b => b.type == "text").map(b => b.text).join("")
            openai.messages.push({role: "system", content: text})
    
    // Messages
    for msg in anthropicPayload.messages:
        if typeof msg.content == string:
            openai.messages.push({role: msg.role, content: msg.content})
            continue
        
        // Array content — process each block
        openContent = []
        toolCalls = []
        
        for block in msg.content:
            switch block.type:
                case "text":
                    openContent.push({type: "text", text: block.text})
                
                case "thinking", "redacted_thinking":
                    skip  // Upstream doesn't support thinking
                
                case "image":
                    if block.source.type == "base64":
                        url = "data:{block.source.media_type};base64,{block.source.data}"
                        openContent.push({type: "image_url", image_url: {url}})
                    else if block.source.type == "url":
                        openContent.push({type: "image_url", image_url: {url: block.source.url}})
                
                case "tool_use":
                    toolCalls.push({
                        id: block.id,
                        type: "function",
                        function: {name: block.name, arguments: JSON.stringify(block.input)}
                    })
                
                case "tool_result":
                    // Tool results become separate messages
                    resultContent = ""
                    if typeof block.content == string:
                        resultContent = block.content
                    else if typeof block.content == array:
                        resultContent = block.content.filter(b => b.text).map(b => b.text).join("")
                    openai.messages.push({
                        role: "tool",
                        tool_call_id: block.tool_use_id,
                        content: resultContent
                    })
        
        // Build the message
        if openContent.length > 0 or toolCalls.length > 0:
            msgBody = {role: msg.role}
            
            if openContent.length > 0:
                // Simplify: single text block with no images → plain string
                if openContent.length == 1 and openContent[0].type == "text":
                    msgBody.content = openContent[0].text
                else:
                    msgBody.content = openContent
            
            if toolCalls.length > 0:
                msgBody.tool_calls = toolCalls
            
            openai.messages.push(msgBody)
    
    // Tools
    if anthropicPayload.tools:
        openai.tools = anthropicPayload.tools.map(t => ({
            type: "function",
            function: {
                name: t.name,
                description: t.description,
                parameters: t.input_schema
            }
        }))
        
        // Tool choice mapping
        if anthropicPayload.tool_choice:
            switch anthropicPayload.tool_choice.type:
                case "auto": openai.tool_choice = "auto"
                case "any": openai.tool_choice = "required"
                case "none": openai.tool_choice = "none"
                case "tool":
                    openai.tool_choice = {
                        type: "function",
                        function: {name: anthropicPayload.tool_choice.name}
                    }
    
    return openai
```

---

## OpenAI to Anthropic Response

```pseudocode
function openaiSyncToAnthropicJSON(openaiResp, origModel, thinkingEnabled):
    msgID = "msg_" + randomHex(12)
    
    choice = openaiResp.choices[0] ?? {}
    message = choice.message ?? {}
    content = []
    
    // Thinking block (if upstream provided reasoning_content)
    if thinkingEnabled and message.reasoning_content:
        content.push({type: "thinking", thinking: message.reasoning_content, signature: ""})
    
    // Text content
    if message.content:
        content.push({type: "text", text: message.content})
    
    // Tool calls
    if message.tool_calls:
        for tc in message.tool_calls:
            input = JSON.parse(tc.function.arguments) ?? {}
            content.push({
                type: "tool_use",
                id: tc.id,
                name: tc.function.name,
                input: input
            })
    
    // Stop reason mapping
    stopReason = "end_turn"
    switch choice.finish_reason:
        case "tool_calls": stopReason = "tool_use"
        case "length": stopReason = "max_tokens"
        case "stop": stopReason = "end_turn"
    
    return {
        id: msgID,
        type: "message",
        role: "assistant",
        content: content,
        model: origModel,
        stop_reason: stopReason,
        stop_sequence: null,
        usage: {
            input_tokens: openaiResp.usage?.prompt_tokens ?? 0,
            output_tokens: openaiResp.usage?.completion_tokens ?? 0,
            cache_creation_input_tokens: 0,
            cache_read_input_tokens: 0
        }
    }
```

---

## OpenAI Stream to Anthropic Stream

This is the most complex conversion — translating OpenAI streaming chunks into Anthropic SSE events in real-time.

```pseudocode
function streamOpenAIToAnthropic(upstreamSSE, writer, origModel, thinkingEnabled):
    msgID = "msg_" + randomHex(12)
    
    // State tracking
    blockIndex = 0
    textBlockStarted = false
    reasoningBlockStarted = false
    toolCallBlockStarted = {}    // openai_tc_index → bool
    toolCallToBlock = {}         // openai_tc_index → anthropic_block_index
    finishReason = ""
    
    // Send initial events
    emit("message_start", {
        type: "message_start",
        message: {
            id: msgID, type: "message", role: "assistant",
            content: [], model: origModel,
            stop_reason: null, stop_sequence: null,
            usage: {input_tokens: 0, output_tokens: 0,
                    cache_creation_input_tokens: 0, cache_read_input_tokens: 0}
        }
    })
    emit("ping", {type: "ping"})
    
    // Process each upstream chunk
    for chunk in upstreamSSE:
        delta = chunk.choices[0].delta
        
        // Capture finish_reason
        if chunk.choices[0].finish_reason:
            finishReason = chunk.choices[0].finish_reason
        
        // Handle reasoning_content (thinking)
        if delta.reasoning_content and thinkingEnabled:
            if not reasoningBlockStarted:
                emit("content_block_start", {
                    type: "content_block_start", index: blockIndex,
                    content_block: {type: "thinking", thinking: "", signature: ""}
                })
                reasoningBlockStarted = true
            emit("content_block_delta", {
                type: "content_block_delta", index: blockIndex,
                delta: {type: "thinking_delta", thinking: delta.reasoning_content}
            })
        
        // Handle text content
        if delta.content:
            // Close reasoning block first
            if reasoningBlockStarted and not textBlockStarted:
                emit("content_block_delta", {
                    type: "content_block_delta", index: blockIndex,
                    delta: {type: "signature_delta", signature: ""}
                })
                emit("content_block_stop", {type: "content_block_stop", index: blockIndex})
                blockIndex++
            
            if not textBlockStarted:
                emit("content_block_start", {
                    type: "content_block_start", index: blockIndex,
                    content_block: {type: "text", text: ""}
                })
                textBlockStarted = true
            
            emit("content_block_delta", {
                type: "content_block_delta", index: blockIndex,
                delta: {type: "text_delta", text: delta.content}
            })
        
        // Handle tool calls
        if delta.tool_calls:
            for tc in delta.tool_calls:
                tcIdx = tc.index
                
                if not toolCallBlockStarted[tcIdx]:
                    // Close text block before first tool call
                    if textBlockStarted and toolCallBlockStarted.isEmpty():
                        emit("content_block_stop", {index: blockIndex})
                        blockIndex++
                    
                    toolCallToBlock[tcIdx] = blockIndex
                    emit("content_block_start", {
                        type: "content_block_start", index: blockIndex,
                        content_block: {
                            type: "tool_use", id: tc.id,
                            name: tc.function.name, input: {}
                        }
                    })
                    toolCallBlockStarted[tcIdx] = true
                    
                    if tc.function.arguments:
                        emit("content_block_delta", {
                            index: blockIndex,
                            delta: {type: "input_json_delta", partial_json: tc.function.arguments}
                        })
                else:
                    bi = toolCallToBlock[tcIdx]
                    if tc.function.arguments:
                        emit("content_block_delta", {
                            index: bi,
                            delta: {type: "input_json_delta", partial_json: tc.function.arguments}
                        })
    
    // Close all open blocks
    if reasoningBlockStarted and not textBlockStarted and toolCallBlockStarted.isEmpty():
        emit("content_block_delta", {index: blockIndex, delta: {type: "signature_delta", signature: ""}})
        emit("content_block_stop", {index: blockIndex})
    
    if textBlockStarted and toolCallBlockStarted.isEmpty():
        emit("content_block_stop", {index: blockIndex})
    
    for tcIdx in toolCallBlockStarted:
        emit("content_block_stop", {index: toolCallToBlock[tcIdx]})
    
    // Stop reason mapping
    stopReason = "end_turn"
    switch finishReason:
        case "tool_calls": stopReason = "tool_use"
        case "length": stopReason = "max_tokens"
    
    emit("message_delta", {
        type: "message_delta",
        delta: {stop_reason: stopReason},
        usage: {output_tokens: 0}
    })
    emit("message_stop", {type: "message_stop"})
```

---

## Responses to Chat Completions Request

```pseudocode
function translateResponsesInput(responsesPayload):
    chatPayload = {
        model: responsesPayload.model,
        stream: false,
        max_tokens: responsesPayload.max_output_tokens ?? 4096
    }
    
    if responsesPayload.temperature: chatPayload.temperature = responsesPayload.temperature
    if responsesPayload.top_p: chatPayload.top_p = responsesPayload.top_p
    
    messages = []
    
    // Instructions → system message
    if responsesPayload.instructions:
        messages.push({role: "system", content: responsesPayload.instructions})
    
    // Input handling
    if typeof responsesPayload.input == string:
        messages.push({role: "user", content: responsesPayload.input})
    
    else if typeof responsesPayload.input == array:
        for item in responsesPayload.input:
            role = item.role
            
            if typeof item.content == string:
                messages.push({role, content: item.content})
            
            else if typeof item.content == array:
                // Flatten input_text/output_text blocks to plain text
                textParts = []
                for block in item.content:
                    if block.type in ["input_text", "output_text", "text"]:
                        textParts.push(block.text)
                if textParts.length > 0:
                    messages.push({role, content: textParts.join("\n")})
    
    chatPayload.messages = messages
    
    // Tools passthrough
    if responsesPayload.tools: chatPayload.tools = responsesPayload.tools
    if responsesPayload.tool_choice: chatPayload.tool_choice = responsesPayload.tool_choice
    
    return chatPayload
```

---

## Chat Completions Stream to Responses Stream

```pseudocode
function streamChatToResponses(upstreamSSE, writer, origModel):
    respID = "resp_" + randomHex(12)
    msgID = "msg_" + randomHex(12)
    seqNum = 0
    created = now()
    
    // Helper: emit with auto-incrementing sequence_number
    function emit(eventType, data):
        data.sequence_number = seqNum++
        writeSSE(writer, eventType, data)
        flush()
    
    // Build the "in progress" response object with all required fields
    inProgressResp = {
        id: respID, object: "response", created_at: created,
        status: "in_progress", model: origModel,
        output: [], usage: null,
        error: null, incomplete_details: null, instructions: null,
        metadata: {}, parallel_tool_calls: true,
        temperature: 1.0, tool_choice: "auto", tools: [],
        top_p: 1.0, max_output_tokens: null, previous_response_id: null,
        reasoning: {effort: null, summary: null},
        store: true, truncation: "disabled", user: null
    }
    
    emit("response.created", {type: "response.created", response: inProgressResp})
    emit("response.in_progress", {type: "response.in_progress", response: inProgressResp})
    
    emit("response.output_item.added", {
        type: "response.output_item.added", output_index: 0,
        item: {type: "message", id: msgID, status: "in_progress",
               role: "assistant", content: []}
    })
    
    emit("response.content_part.added", {
        type: "response.content_part.added",
        item_id: msgID, output_index: 0, content_index: 0,
        part: {type: "output_text", text: "", annotations: []}
    })
    
    // Track state
    fullText = ""
    textBlockSent = false
    toolCallNames = {}  // index → name
    toolCallIDs = {}    // index → id
    toolCallArgs = {}   // index → accumulated arguments
    upstreamUsage = {input_tokens: 0, output_tokens: 0, total_tokens: 0}
    
    for chunk in upstreamSSE:
        // Capture usage from final chunk
        if chunk.usage:
            upstreamUsage = {
                input_tokens: chunk.usage.prompt_tokens,
                output_tokens: chunk.usage.completion_tokens,
                total_tokens: chunk.usage.total_tokens
            }
        
        delta = chunk.choices[0]?.delta
        if not delta: continue
        
        if delta.content:
            fullText += delta.content
            textBlockSent = true
            emit("response.output_text.delta", {
                type: "response.output_text.delta",
                item_id: msgID, output_index: 0, content_index: 0,
                delta: delta.content
            })
        
        // Accumulate tool calls
        if delta.tool_calls:
            for tc in delta.tool_calls:
                idx = tc.index
                if tc.id: toolCallIDs[idx] = tc.id
                if tc.function?.name: toolCallNames[idx] = tc.function.name
                if tc.function?.arguments: toolCallArgs[idx] += tc.function.arguments
    
    // Close text block
    allOutput = []
    outputIndex = 0
    
    if textBlockSent or fullText:
        emit("response.output_text.done", {
            type: "response.output_text.done",
            item_id: msgID, output_index: 0, content_index: 0,
            text: fullText
        })
        emit("response.content_part.done", {
            type: "response.content_part.done",
            item_id: msgID, output_index: 0, content_index: 0,
            part: {type: "output_text", text: fullText, annotations: []}
        })
        msgItem = {
            type: "message", id: msgID, status: "completed",
            role: "assistant",
            content: [{type: "output_text", text: fullText, annotations: []}]
        }
        emit("response.output_item.done", {
            type: "response.output_item.done", output_index: outputIndex, item: msgItem
        })
        allOutput.push(msgItem)
        outputIndex++
    
    // Emit function_call items
    for i in sorted(toolCallNames.keys()):
        fcID = "fc_" + randomHex(12)
        fcItem = {
            type: "function_call", id: fcID,
            call_id: toolCallIDs[i], name: toolCallNames[i],
            arguments: toolCallArgs[i], status: "completed"
        }
        emit("response.output_item.added", {
            type: "response.output_item.added", output_index: outputIndex, item: fcItem
        })
        emit("response.function_call_arguments.done", {
            type: "response.function_call_arguments.done",
            item_id: fcID, output_index: outputIndex,
            arguments: toolCallArgs[i]
        })
        emit("response.output_item.done", {
            type: "response.output_item.done", output_index: outputIndex, item: fcItem
        })
        allOutput.push(fcItem)
        outputIndex++
    
    // If nothing output, send empty message
    if allOutput.isEmpty():
        emptyItem = {type: "message", id: msgID, status: "completed",
                     role: "assistant",
                     content: [{type: "output_text", text: "", annotations: []}]}
        emit("response.output_item.done", {output_index: 0, item: emptyItem})
        allOutput.push(emptyItem)
    
    // Final event
    completedResp = copy(inProgressResp)
    completedResp.status = "completed"
    completedResp.output = allOutput
    completedResp.usage = upstreamUsage
    
    emit("response.completed", {type: "response.completed", response: completedResp})
```

---

## Chat Completions to Responses Response

Non-streaming conversion:

```pseudocode
function openaiChatToResponsesJSON(openaiResp, origModel):
    respID = "resp_" + randomHex(12)
    msgID = "msg_" + randomHex(12)
    
    choice = openaiResp.choices[0]
    message = choice?.message ?? {}
    text = message.content ?? ""
    
    output = []
    
    // Text message
    if text:
        output.push({
            type: "message", id: msgID, status: "completed",
            role: "assistant",
            content: [{type: "output_text", text: text, annotations: []}]
        })
    
    // Function calls
    if message.tool_calls:
        for tc in message.tool_calls:
            output.push({
                type: "function_call",
                id: "fc_" + randomHex(12),
                call_id: tc.id,
                name: tc.function.name,
                arguments: tc.function.arguments,
                status: "completed"
            })
    
    // Empty fallback
    if output.isEmpty():
        output.push({
            type: "message", id: msgID, status: "completed",
            role: "assistant",
            content: [{type: "output_text", text: "", annotations: []}]
        })
    
    return {
        id: respID,
        object: "response",
        created_at: openaiResp.created ?? now(),
        status: "completed",
        model: origModel,
        output: output,
        usage: {
            input_tokens: openaiResp.usage?.prompt_tokens ?? 0,
            output_tokens: openaiResp.usage?.completion_tokens ?? 0,
            total_tokens: openaiResp.usage?.total_tokens ?? 0
        }
    }
```

---

## Tool Format Conversion

### Anthropic tools → OpenAI tools

```
Anthropic:
{name: "get_weather", description: "Get weather", input_schema: {type: "object", properties: {...}}}

OpenAI:
{type: "function", function: {name: "get_weather", description: "Get weather", parameters: {type: "object", properties: {...}}}}
```

Mapping: `input_schema` → `function.parameters`

### OpenAI tool_calls → Anthropic tool_use

```
OpenAI (in message):
{tool_calls: [{id: "call_abc", type: "function", function: {name: "get_weather", arguments: '{"location":"SF"}'}}]}

Anthropic (in content):
{type: "tool_use", id: "call_abc", name: "get_weather", input: {location: "SF"}}
```

Mapping: `JSON.parse(arguments)` → `input`

### Anthropic tool_result → OpenAI tool message

```
Anthropic (in content):
{type: "tool_result", tool_use_id: "call_abc", content: "72°F and sunny"}

OpenAI (separate message):
{role: "tool", tool_call_id: "call_abc", content: "72°F and sunny"}
```

### Anthropic tool_choice → OpenAI tool_choice

| Anthropic | OpenAI |
|-----------|--------|
| `{type: "auto"}` | `"auto"` |
| `{type: "any"}` | `"required"` |
| `{type: "none"}` | `"none"` |
| `{type: "tool", name: "X"}` | `{type: "function", function: {name: "X"}}` |
