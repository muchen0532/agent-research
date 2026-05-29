# Tau2-Bench Model Selection Notes | Tau2-Bench 模型选型记录

## Background | 背景

While evaluating open-source models on Tau2-Bench, I tested multiple Qwen and DeepSeek models through LiteLLM-compatible providers.

在 Tau2-Bench 测试过程中，我通过 LiteLLM 兼容接口测试了多个 Qwen 和 DeepSeek 模型。

Several compatibility and deployment issues were encountered during evaluation.

在评测过程中遇到了一些兼容性和部署方面的问题。

---

## Qwen2.5-14B-Instruct

### Issue | 问题

The model returns `ToolCall.arguments` as a JSON string instead of a dictionary object.

模型返回的 `ToolCall.arguments` 为 JSON 字符串，而非字典对象。

Example | 示例：

```json
{
  "arguments": "{\"product_id\":\"1656367028\"}"
}
```

Tau2-Bench expects a dictionary:

Tau2-Bench 期望的格式：

```json
{
  "arguments": {
    "product_id": "1656367028"
  }
}
```

This causes a validation error: `Input should be a valid dictionary`

导致校验错误：`Input should be a valid dictionary`

### Status | 结论

Not recommended without additional argument-parsing middleware.

除非增加额外的参数解析层，否则不推荐使用。

---

## Qwen2.5-32B-Instruct

### Issue | 问题

Tool calling works correctly, but TPM limits (40,000 TPM on SiliconFlow) are frequently exceeded during benchmark execution. A single retail domain task can consume over 40,000 tokens due to the lengthy system policy prompt, causing mid-conversation rate limit errors.

模型 Tool Calling 正常，但在 SiliconFlow 上 TPM 上限为 40,000，retail domain 单个任务因 system prompt 较长，单次对话内 token 消耗即可超出限制，导致对话中途限流报错。

### Status | 结论

Usable for small-scale tests but not suitable for full benchmark runs.

小规模测试可用，不适合完整 Benchmark 运行。

---

## Qwen2.5-72B-Instruct

### Issue | 问题

Same TPM issue as the 32B version. Model capability is stronger but the rate limit bottleneck remains unchanged on SiliconFlow.

与 32B 版本相同的 TPM 问题。模型能力更强，但在 SiliconFlow 上的限速瓶颈没有改善。

### Status | 结论

Usable but not ideal for large-scale benchmark runs.

可以使用，但不适合作为大规模 Benchmark 的默认选择。

---

## Qwen3-14B (SiliconFlow)

### Issue | 问题

Tool calling works correctly and produces real `tool_calls` objects (verified on retail domain tasks). However, the same 40,000 TPM limit applies — a single retail task exceeds this within the conversation, causing rate limit failures.

Tool Calling 正常，在 retail domain 任务中可产生真实的 `tool_calls` 对象（已验证）。但同样受 SiliconFlow 40,000 TPM 限制影响，单个 retail 任务在对话内即触发限速。

Additionally, Qwen3 models enable thinking mode by default. This must be disabled explicitly to avoid `reasoning_content` fields that cause LiteLLM multi-turn errors.

另外，Qwen3 系列默认开启思考模式，需显式关闭，否则 `reasoning_content` 字段会导致 LiteLLM 多轮对话报错。

### Status | 结论

Functionally capable but blocked by TPM limits on SiliconFlow. May be viable with a higher-tier account or alternative provider.

功能上可用，但受 SiliconFlow TPM 限制无法稳定运行完整实验。若使用更高配额账户或其他平台，可能可行。

---

## DeepSeek-V3 (SiliconFlow, Third-Party)

### Issue | 问题

DeepSeek-V3 in non-thinking mode has unreliable tool calling support. According to official documentation, stable tool calling under thinking mode was introduced from V3.2 onwards. In practice, the model frequently expressed tool-use intent as markdown text rather than emitting actual `tool_calls` objects, causing the agent to produce zero real tool calls across multiple retail and telecom domain tasks.

DeepSeek-V3 非思考模式下 Tool Calling 支持不稳定。官方文档说明从 V3.2 起才在思考模式下支持工具调用。实测中，模型频繁将工具调用意图输出为 markdown 文本而非真实的 `tool_calls` 对象，在 retail 和 telecom 多个任务中 agent 侧实际 tool call 数量为零。

This is a model-level limitation, not a provider issue.

这是模型本身的限制，与第三方平台无关。

### Status | 结论

Not selected. The agent produces no real tool calls, making governor interception experiments impossible.

未采用。agent 无法产生真实 tool call，governor 拦截实验无法进行。

---

## Final Choice | 最终选择

Selected model | 最终采用：

```
deepseek/deepseek-v4-flash
```

Accessed via the official DeepSeek API (`https://api.deepseek.com`), not through third-party providers.

通过 DeepSeek 官方 API（`https://api.deepseek.com`）调用，非第三方平台。

Reasons | 原因：

- Stable tool calling behavior verified on retail and telecom domains
- 在 retail 和 telecom domain 均验证了稳定的 Tool Calling 行为
- No TPM bottleneck observed during benchmark runs
- Benchmark 运行中未观察到 TPM 限速问题
- Thinking mode can be disabled via `extra_body`, avoiding `reasoning_content` LiteLLM errors
- 可通过 `extra_body` 关闭思考模式，避免 `reasoning_content` 字段导致 LiteLLM 多轮对话报错
- Official API with consistent behavior
- 官方 API，行为一致性好

---

## Recommended Configuration | 推荐配置

`.env`:

```
OPENAI_API_KEY=<your-deepseek-api-key>
OPENAI_BASE_URL=https://api.deepseek.com
```

Agent:

```bash
--agent-llm deepseek/deepseek-v4-flash \
--agent-llm-args '{"temperature": 0.0, "extra_body": {"thinking": {"type": "disabled"}}}'
```

User simulator:

```bash
--user-llm deepseek/deepseek-v4-flash \
--user-llm-args '{"temperature": 0.0, "extra_body": {"thinking": {"type": "disabled"}}}'
```

The `thinking: disabled` flag is required to suppress the `reasoning_content` field, which causes LiteLLM to raise errors in multi-turn conversations.

`thinking: disabled` 是必要配置，用于关闭思考模式，避免 `reasoning_content` 字段在 LiteLLM 多轮对话中触发报错。

---

## Summary | 总结

| Model | Result | Main Issue |
|---|---|---|
| Qwen2.5-14B-Instruct | Not Recommended | `ToolCall.arguments` type mismatch (string vs dict) |
| Qwen2.5-32B-Instruct | Conditional | TPM limits on SiliconFlow (40K TPM) |
| Qwen2.5-72B-Instruct | Conditional | TPM limits on SiliconFlow (40K TPM) |
| Qwen3-14B (SiliconFlow) | Conditional | TPM limits; thinking mode must be disabled |
| DeepSeek-V3 (Third-Party) | Not Recommended | Model-level tool calling instability in non-thinking mode |
| DeepSeek-V4-Flash (Official) | **Recommended** | Stable benchmark execution |
