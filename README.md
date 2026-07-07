# llm-api-adapter

构建 LLM API 代理/适配层的实战参考技能。将任意 LLM 后端暴露为三种业界标准 API 接口，语言无关（Go / Python / TypeScript / Rust 均适用）。

## 覆盖接口

- **OpenAI Chat Completions** (`POST /v1/chat/completions`)
- **Anthropic Messages** (`POST /v1/messages`)
- **OpenAI Responses** (`POST /v1/responses`)

## 核心内容

- 三种协议的完整请求 / SSE 流式格式（含完整事件序列）
- 格式互转规则（Anthropic ↔ OpenAI、Responses ↔ Chat）
- 账号池三级错误分类与重试循环
- Anthropic 兼容性边缘场景（孤立 tool_use、cache_control 剥离、thinking 剥离、图片格式转换）
- OpenAI Responses 严格字段（`sequence_number` / `item_id` / `annotations`）
- 严格客户端空响应调试清单（Cherry Studio / Cursor / Claude Code）
- 模型名称映射与实现检查清单

## 作为 Agent Skill 使用

```bash
skills add https://github.com/Xichun123/llm-api-adapter --skill llm-api-adapter -g -a universal -y
```

## License

MIT
