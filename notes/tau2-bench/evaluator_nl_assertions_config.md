# Tau2-Bench NL Assertions Evaluator Config | NL 断言评估器配置问题记录

## Background | 背景

Tau2-Bench includes an LLM-as-Judge evaluator for tasks that have `nl_assertions` in their evaluation criteria. This evaluator calls an external LLM to verify whether the agent's response satisfies natural-language assertions such as "Agent should tell the user that there are 10 t-shirt options available."

Tau2-Bench 对包含 `nl_assertions` 的任务使用 LLM-as-Judge 进行评估。该评估器会调用外部 LLM，判断 agent 的回复是否满足自然语言断言，例如"Agent 应告知用户当前有 10 种 T 恤可选"。

---

## Issue | 问题

When running Tau2-Bench with a non-OpenAI provider (e.g., DeepSeek official API), tasks with `nl_assertions` fail with the following error:

使用非 OpenAI 提供商（例如 DeepSeek 官方 API）运行 Tau2-Bench 时，包含 `nl_assertions` 的任务会报错：

```
litellm.BadRequestError: OpenAIException - The supported API model names are
deepseek-v4-pro or deepseek-v4-flash, but you passed gpt-4.1-2025-04-14.
```

The root cause is that the NL assertions evaluator model is hardcoded in `src/tau2/config.py`:

根本原因是 NL 断言评估器的模型名称在 `src/tau2/config.py` 中被硬编码：

```python
DEFAULT_LLM_NL_ASSERTIONS = "gpt-4.1-2025-04-14"
DEFAULT_LLM_NL_ASSERTIONS_TEMPERATURE = 0.0
DEFAULT_LLM_NL_ASSERTIONS_ARGS = {"temperature": DEFAULT_LLM_NL_ASSERTIONS_TEMPERATURE}
```

This value is imported directly in `src/tau2/evaluator/evaluator_nl_assertions.py`:

该值被直接导入 `src/tau2/evaluator/evaluator_nl_assertions.py`：

```python
from tau2.config import DEFAULT_LLM_NL_ASSERTIONS, DEFAULT_LLM_NL_ASSERTIONS_ARGS
```

---

## Why It Cannot Be Overridden via CLI | 为什么无法通过命令行覆盖

The `tau2 run` CLI exposes `--agent-llm` and `--user-llm` parameters, which override `DEFAULT_LLM_AGENT` and `DEFAULT_LLM_USER` respectively. However, there is no corresponding `--judge-llm` or `--nl-assertions-llm` parameter for the evaluator model.

`tau2 run` 命令行提供了 `--agent-llm` 和 `--user-llm` 参数，分别对应 `DEFAULT_LLM_AGENT` 和 `DEFAULT_LLM_USER`。但评估器模型没有对应的 CLI 参数，例如 `--judge-llm` 或 `--nl-assertions-llm`。

Environment variables in `.env` also have no effect, as `config.py` does not read from the environment for this value.

`.env` 中设置环境变量同样无效，因为 `config.py` 对该值未实现环境变量读取。

---

## Affected Tasks | 受影响的任务

Only tasks where `reward_basis` includes `NL_ASSERTION` are affected. In the retail domain, this includes tasks 2, 3, and 4 (tasks that ask the agent to count available product variants and report the number).

只有 `reward_basis` 包含 `NL_ASSERTION` 的任务会受影响。在 retail domain 中，这包括任务 2、3、4（要求 agent 统计并告知可用商品变体数量的任务）。

Tasks where `reward_basis` is `["DB"]` or `["DB", "NL_ASSERTION"]` with empty `nl_assertions` are not affected.

`reward_basis` 为 `["DB"]` 或 `nl_assertions` 为空的任务不受影响。

---

## Fix | 解决方法

Edit `src/tau2/config.py` directly. Change the following lines:

直接修改 `src/tau2/config.py`，将以下内容：

```python
DEFAULT_LLM_NL_ASSERTIONS = "gpt-4.1-2025-04-14"
DEFAULT_LLM_NL_ASSERTIONS_TEMPERATURE = 0.0
DEFAULT_LLM_NL_ASSERTIONS_ARGS = {"temperature": DEFAULT_LLM_NL_ASSERTIONS_TEMPERATURE}
```

To match your provider and model. For DeepSeek:

替换为你使用的提供商和模型。以 DeepSeek 为例：

```python
DEFAULT_LLM_NL_ASSERTIONS = "deepseek/deepseek-v4-flash"
DEFAULT_LLM_NL_ASSERTIONS_TEMPERATURE = 0.0
DEFAULT_LLM_NL_ASSERTIONS_ARGS = {
    "temperature": DEFAULT_LLM_NL_ASSERTIONS_TEMPERATURE,
    "extra_body": {"thinking": {"type": "disabled"}},
}
```

Note: The `extra_body` with `thinking: disabled` is required for DeepSeek V4 Flash to suppress the `reasoning_content` field, which causes LiteLLM multi-turn errors.

注意：DeepSeek V4 Flash 需要通过 `extra_body` 关闭思考模式，否则 `reasoning_content` 字段会导致 LiteLLM 多轮对话报错。

---

## Also Check | 同时检查

The same `config.py` file also hardcodes the default agent and user LLM:

同一个 `config.py` 文件中，agent 和 user 的默认模型也是硬编码的：

```python
DEFAULT_LLM_AGENT = "gpt-4.1-2025-04-14"
DEFAULT_LLM_USER = "gpt-4.1-2025-04-14"
```

These can be overridden via `--agent-llm` and `--user-llm` on the CLI, so changing them in `config.py` is optional. However, if you always use the same non-OpenAI model, updating the defaults here avoids having to pass the flags every time.

这两个值可以通过 `--agent-llm` 和 `--user-llm` 命令行参数覆盖，因此修改 `config.py` 是可选的。但如果你固定使用某个非 OpenAI 模型，在此修改默认值可以省去每次传参的麻烦。

---

## Summary | 总结

| Item | Detail |
|---|---|
| Affected file | `src/tau2/config.py` |
| Hardcoded value | `DEFAULT_LLM_NL_ASSERTIONS = "gpt-4.1-2025-04-14"` |
| CLI override available | No |
| Env var override available | No |
| Fix | Edit `config.py` directly |
| Affected tasks | Tasks with non-empty `nl_assertions` in evaluation criteria |
| DeepSeek extra config | Add `extra_body: {thinking: {type: disabled}}` to `DEFAULT_LLM_NL_ASSERTIONS_ARGS` |
