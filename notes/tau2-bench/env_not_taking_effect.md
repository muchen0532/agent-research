# Tau2-Bench .env Not Taking Effect | .env 文件不生效问题记录

## Background | 背景

Tau2-Bench reads API keys and base URLs from a `.env` file via `python-dotenv`. However, `.env` values can be silently overridden by system-level environment variables, causing unexpected behavior that is difficult to debug.

Tau2-Bench 通过 `python-dotenv` 从 `.env` 文件读取 API key 和 base URL。但 `.env` 中的值可能被系统级环境变量静默覆盖，导致难以排查的异常行为。

---

## Issue | 问题

After updating `.env` with a new `OPENAI_BASE_URL` or `OPENAI_API_KEY`, the benchmark continues to use the old value. Requests are still sent to the previous endpoint, or authentication fails despite the key appearing correct in `.env`.

在 `.env` 中更新 `OPENAI_BASE_URL` 或 `OPENAI_API_KEY` 后，Benchmark 仍然使用旧值。请求依然发往旧的 endpoint，或者尽管 `.env` 中的 key 看起来正确，认证仍然失败。

---

## Root Cause | 根本原因

`python-dotenv` does **not** override environment variables that are already set in the current shell session. If `OPENAI_API_KEY` or `OPENAI_BASE_URL` was previously exported in the shell (e.g., via a prior `$env:OPENAI_API_KEY = ...` command in PowerShell, or `export` in bash), those values take precedence over `.env`.

`python-dotenv` **不会**覆盖当前 shell 会话中已存在的环境变量。如果 `OPENAI_API_KEY` 或 `OPENAI_BASE_URL` 之前已通过 shell 命令导出（例如 PowerShell 中的 `$env:OPENAI_API_KEY = ...`，或 bash 中的 `export`），这些值的优先级高于 `.env`。

Priority order (highest to lowest) | 优先级顺序（从高到低）：

```
Shell environment variables  >  .env file  >  default values
系统环境变量                 >  .env 文件  >  默认值
```

---

## Diagnosis | 排查方法

Check whether the variable is set at the shell level before blaming `.env`:

在认为是 `.env` 问题之前，先检查 shell 层面是否已设置该变量：

**PowerShell:**

```powershell
echo $env:OPENAI_API_KEY
echo $env:OPENAI_BASE_URL
```

**bash / zsh:**

```bash
echo $OPENAI_API_KEY
echo $OPENAI_BASE_URL
```

If these commands return a value, the shell-level variable is overriding `.env`.

如果这些命令有输出，说明 shell 级变量正在覆盖 `.env`。

---

## Fix | 解决方法

Remove the shell-level variable so `.env` can take effect.

删除 shell 级变量，使 `.env` 生效。

**PowerShell:**

```powershell
Remove-Item Env:OPENAI_API_KEY -ErrorAction SilentlyContinue
Remove-Item Env:OPENAI_BASE_URL -ErrorAction SilentlyContinue
```

**bash / zsh:**

```bash
unset OPENAI_API_KEY
unset OPENAI_BASE_URL
```

After removing them, verify `.env` is being read correctly by running a quick test call or checking logs.

删除后，通过一次测试调用或查看日志确认 `.env` 已被正确读取。

---

## Prevention | 预防措施

Avoid setting `OPENAI_API_KEY` or `OPENAI_BASE_URL` as persistent shell variables (e.g., in `.bashrc`, `.zshrc`, or PowerShell profile). Manage all provider credentials exclusively through `.env` to keep configuration in one place.

避免将 `OPENAI_API_KEY` 或 `OPENAI_BASE_URL` 设置为持久化的 shell 变量（例如写入 `.bashrc`、`.zshrc` 或 PowerShell profile）。所有提供商凭据统一通过 `.env` 管理，避免多处配置冲突。

When switching providers (e.g., from SiliconFlow to DeepSeek official API), always check for stale shell variables first.

切换提供商时（例如从 SiliconFlow 切换到 DeepSeek 官方 API），务必先检查 shell 中是否有残留的旧变量。

---

## Summary | 总结

| Item | Detail |
|---|---|
| Affected variables | `OPENAI_API_KEY`, `OPENAI_BASE_URL` |
| Root cause | Shell environment variables take priority over `.env` |
| Diagnosis | `echo $env:OPENAI_BASE_URL` (PowerShell) or `echo $OPENAI_BASE_URL` (bash) |
| Fix | `Remove-Item Env:OPENAI_BASE_URL` (PowerShell) or `unset OPENAI_BASE_URL` (bash) |
| Prevention | Never set these as persistent shell variables; use `.env` exclusively |
