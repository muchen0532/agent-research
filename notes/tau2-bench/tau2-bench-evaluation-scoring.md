# tau2-bench 评分逻辑源码说明

本文整理 `tau2-bench` 中一次仿真如何得到 `reward`，以及多次任务运行如何聚合成 `avg_reward`、`pass^k` 等指标。

## 总览

评分逻辑分两层：

1. 单条 simulation 的评分：把一条轨迹和任务定义比较，得到 `RewardInfo.reward`。
2. 实验结果聚合：把多个 simulation 的 reward 汇总成 `avg_reward`、`pass^1`、`pass^k` 等指标。

对应源码：

- 单条评分入口：`tau2-bench/src/tau2/evaluator/evaluator.py`
- DB / 环境状态评分：`tau2-bench/src/tau2/evaluator/evaluator_env.py`
- action 匹配评分：`tau2-bench/src/tau2/evaluator/evaluator_action.py`
- communicate 信息评分：`tau2-bench/src/tau2/evaluator/evaluator_communicate.py`
- NL assertion 评分：`tau2-bench/src/tau2/evaluator/evaluator_nl_assertions.py`
- 指标聚合：`tau2-bench/src/tau2/metrics/agent_metrics.py`
- 评分说明文档：`tau2-bench/docs/evaluation.md`

## 单条 Simulation 评分入口

核心函数：

```python
evaluate_simulation(...)
```

位置：

```text
tau2-bench/src/tau2/evaluator/evaluator.py
```

这个函数负责：

1. 检查 simulation 是否正常结束。
2. 根据 `EvaluationType` 选择要运行的 evaluator。
3. 运行环境评分、动作评分、沟通评分、NL assertion 评分。
4. 根据任务里的 `evaluation_criteria.reward_basis` 组合最终 reward。

如果 simulation 的 `termination_reason` 不是 `AGENT_STOP` 或 `USER_STOP`，会直接返回：

```python
RewardInfo(reward=0.0, reward_basis=None, ...)
```

如果任务没有 `evaluation_criteria`，会返回：

```python
RewardInfo(reward=1.0, reward_basis=None, ...)
```

## EvaluationType

`EvaluationType` 定义在：

```text
tau2-bench/src/tau2/evaluator/evaluator.py
```

主要取值：

- `ENV`：只评估环境 / DB 状态。
- `COMMUNICATE`：只评估 agent 是否说出了要求的信息。
- `ACTION`：只评估 agent 是否调用了指定工具。
- `NL_ASSERTIONS`：只评估自然语言断言。
- `ALL`：运行 ENV、ACTION、COMMUNICATE，并在需要时运行 NL assertion；最终 reward 只使用 `reward_basis` 中指定的组件。
- `ALL_WITH_NL_ASSERTIONS`：类似 `ALL`，但强制运行 NL assertion，用于诊断。
- `ALL_IGNORE_BASIS`：忽略任务的 `reward_basis`，把 ENV、ACTION、COMMUNICATE 都乘进最终分数。
- `ALL_WITH_NL_ASSERTIONS_IGNORE_BASIS`：忽略 `reward_basis`，并额外包含 NL assertion。

默认评估通常使用 `EvaluationType.ALL`。

## reward_basis 如何决定最终分数

任务定义中的关键字段：

```json
"evaluation_criteria": {
  "actions": [...],
  "communicate_info": [...],
  "env_assertions": [...],
  "nl_assertions": [...],
  "reward_basis": ["DB", "COMMUNICATE"]
}
```

最终 reward 是 `reward_basis` 中各组件 reward 的乘积。

例如：

```json
"reward_basis": ["DB", "COMMUNICATE"]
```

则：

```text
final_reward = db_reward * communicate_reward
```

如果 DB 正确但沟通信息缺失：

```text
db_reward = 1.0
communicate_reward = 0.0
final_reward = 0.0
```

如果两者都满足：

```text
db_reward = 1.0
communicate_reward = 1.0
final_reward = 1.0
```

在 `evaluator.py` 中，组合逻辑大致是：

```python
reward = 1.0

if task_reward_basis & env_bases:
    reward *= env_reward_info.reward

if task_reward_basis & action_bases:
    reward *= action_reward_info.reward

if task_reward_basis & nl_bases:
    reward *= nl_reward_info.reward

if task_reward_basis & comm_bases:
    reward *= communicate_reward_info.reward
```

## DB / 环境状态评分

核心类：

```python
EnvironmentEvaluator
```

位置：

```text
tau2-bench/src/tau2/evaluator/evaluator_env.py
```

它的核心逻辑是比较两个环境的最终 DB hash：

1. predicted environment：用 agent 实际运行轨迹重放得到。
2. gold environment：用任务定义中的 `evaluation_criteria.actions` 在新环境中重放得到。
3. 比较两边的 agent DB hash 和 user DB hash。
4. 两者都匹配则 `db_reward = 1.0`，否则 `db_reward = 0.0`。

关键逻辑：

```python
predicted_environment.set_state(
    initialization_data=initialization_data,
    initialization_actions=initialization_actions,
    message_history=list(full_trajectory),
)

gold_environment.set_state(
    initialization_data=initialization_data,
    initialization_actions=initialization_actions,
    message_history=message_history,
)

for action in golden_actions:
    gold_environment.make_tool_call(...)

agent_db_match = agent_db_hash == predicted_agent_db_hash
user_db_match = user_db_hash == predicted_user_db_hash

if agent_db_match and user_db_match:
    db_reward = 1.0
else:
    db_reward = 0.0
```

这里最容易误解的一点是：`evaluation_criteria.actions` 不是默认要求 agent 必须逐个调用的动作清单。它首先用于构造 gold environment 的目标 DB 状态。只要 agent 通过其他路径得到相同 DB 状态，也可以通过 DB 评分。

## action 评分

核心类：

```python
ActionEvaluator
```

位置：

```text
tau2-bench/src/tau2/evaluator/evaluator_action.py
```

它检查任务 `evaluation_criteria.actions` 中列出的每个 action，是否能在 agent 的工具调用轨迹中找到匹配项。

只有当 `RewardType.ACTION` 出现在任务的 `reward_basis` 中时，action 评分才会影响最终 reward。

对 airline、retail、telecom 等常见域，默认 `reward_basis` 通常是：

```json
["DB", "COMMUNICATE"]
```

因此 action 检查即使运行了，通常也只是诊断信息，不直接决定最终分数。

## communicate 评分

核心类：

```python
CommunicateEvaluator
```

位置：

```text
tau2-bench/src/tau2/evaluator/evaluator_communicate.py
```

它检查 `evaluation_criteria.communicate_info` 中要求的信息是否出现在 agent 发给用户的消息中。

如果 `communicate_info` 为空，通常 `communicate_reward = 1.0`。

如果有多个要求，所有要求都满足才是满分；任一项缺失会使对应组件 reward 变成 `0.0`。

## NL assertion 评分

核心类：

```python
NLAssertionsEvaluator
```

位置：

```text
tau2-bench/src/tau2/evaluator/evaluator_nl_assertions.py
```

它用 LLM judge 判断自然语言断言是否满足。

这个部分是实验性逻辑。只有当 `RewardType.NL_ASSERTION` 在 `reward_basis` 中，或者使用强制运行 NL assertion 的 evaluation type 时，才会参与或被记录。

## Full-duplex / Voice 模式

full-duplex 语音模式使用对应的 evaluator：

- `FullDuplexEnvironmentEvaluator`
- `FullDuplexActionEvaluator`
- `FullDuplexCommunicateEvaluator`
- `FullDuplexNLAssertionsEvaluator`

这些类仍在同一批 evaluator 文件中。主要区别是输入轨迹不是普通 `messages`，而是 `ticks`。

在环境评分中，`FullDuplexEnvironmentEvaluator` 会先把 ticks 转成 message history，再执行和普通模式类似的 DB 状态比较。

## Simulation 运行后如何附加 reward_info

单条 simulation 执行和评分连接点：

```text
tau2-bench/src/tau2/runner/simulation.py
```

其中会调用：

```python
reward_info = evaluate_simulation(...)
simulation.reward_info = reward_info
```

批量运行时的相关逻辑在：

```text
tau2-bench/src/tau2/runner/batch.py
```

## pass^k 和 avg_reward 聚合

聚合指标在：

```text
tau2-bench/src/tau2/metrics/agent_metrics.py
```

核心函数：

```python
is_successful(reward)
pass_hat_k(num_trials, success_count, k)
get_tasks_pass_hat_k(results)
compute_metrics(results)
```

### is_successful

```python
def is_successful(reward: float) -> bool:
    return (1 - 1e-6) <= reward <= (1 + 1e-6)
```

也就是说，只有 reward 约等于 `1.0` 才算成功。

### pass_hat_k

```python
def pass_hat_k(num_trials: int, success_count: int, k: int) -> float:
    return math.comb(success_count, k) / math.comb(num_trials, k)
```

含义是：对同一个 task 跑了 `num_trials` 次，其中成功 `success_count` 次，从这些 trial 中抽 `k` 次全部成功的组合比例。

例如：

```text
num_trials = 4
success_count = 2
k = 1
pass^1 = C(2, 1) / C(4, 1) = 0.5
```

```text
num_trials = 4
success_count = 2
k = 2
pass^2 = C(2, 2) / C(4, 2) = 1 / 6
```

### compute_metrics

`compute_metrics(results)` 会：

1. 过滤掉 `TerminationReason.INFRASTRUCTURE_ERROR` 的 simulation。
2. 计算每条 simulation 是否成功。
3. 按 `task_id` 分组计算每个 task 的 `pass^k`。
4. 对所有 task 的 `pass^k` 取平均，得到整体 `pass_hat_ks`。
5. 计算 `avg_reward`、平均成本、DB match 数量、action 诊断统计、termination 统计等。

核心逻辑：

```python
avg_reward = df.reward.mean()

for column in df_pass_hat_k.columns:
    if match := re.match(r"pass\^(\d+)", column):
        k = int(match.group(1))
        pass_hat_ks[k] = df_pass_hat_k[column].mean()
```

## Leaderboard 指标

leaderboard 准备提交时会调用 `compute_metrics(results)`：

```text
tau2-bench/src/tau2/scripts/leaderboard/prepare_submission.py
```

然后把 `metrics.pass_hat_ks` 转成：

```text
pass_1
pass_2
pass_3
pass_4
```

相关字段定义在：

```text
tau2-bench/src/tau2/scripts/leaderboard/submission.py
```

## 官方评分理解

对于 airline、retail、telecom 等主要域，常见最终评分是：

```json
["DB", "COMMUNICATE"]
```

因此重点是：

1. agent 最终是否把环境 DB 改到了正确状态。
2. agent 是否向用户传达了任务要求的信息。

`evaluation_criteria.actions` 常被误读为必须执行的动作列表，但在默认评分中，它更准确的作用是构造 gold DB 状态。agent 可以用不同工具调用顺序、不同查询路径，甚至在某些任务中不调用某些只读工具，只要最终 DB 状态和沟通要求满足，就可以得到满分。

## 源码索引

| 目的 | 文件 |
| --- | --- |
| 单条轨迹评分入口 | `src/tau2/evaluator/evaluator.py` |
| DB / env assertion 评分 | `src/tau2/evaluator/evaluator_env.py` |
| action 匹配评分 | `src/tau2/evaluator/evaluator_action.py` |
| communicate 信息评分 | `src/tau2/evaluator/evaluator_communicate.py` |
| NL assertion 评分 | `src/tau2/evaluator/evaluator_nl_assertions.py` |
| reward 数据结构 | `src/tau2/data_model/simulation.py` |
| task / reward_basis 数据结构 | `src/tau2/data_model/tasks.py` |
| 单条 simulation 运行后评分 | `src/tau2/runner/simulation.py` |
| 批量运行评分 | `src/tau2/runner/batch.py` |
| pass^k / avg_reward 聚合 | `src/tau2/metrics/agent_metrics.py` |
| leaderboard 提交指标 | `src/tau2/scripts/leaderboard/prepare_submission.py` |
| 官方评分说明 | `docs/evaluation.md` |
