---
publish: true
created: 2026-10-30T09:19:07.060Z
modified: 2026-11-02T01:22:47.849Z
---

# Agent 会话管理

Claude Code 和 Codex 的会话通常使用 UUID 标识。`c` 命令可以读取当前会话 ID，生成带可读标题的恢复命令，并复制到 Windows 剪贴板。

# 使用方法

在 Claude Code 或 Codex 中执行 `!c`。
生成结果示例：

```c
claude --resume <session-id> # <pane-title>
```

或：

```c
codex resume <session-id> # <pane-title>
```

# 实现代码

1. 优先读取 `CLAUDE_CODE_SESSION_ID`，使用 `claude --resume`。
2. 没有 Claude Code 会话 ID 时，读取 `CODEX_SESSION_ID`，使用 `codex resume`。
3. 两个变量都不存在时直接报错，不生成无效命令。
4. 尝试读取 tmux pane 标题，清理 ANSI 控制序列、动画字符和无意义标题。
5. 标题只作为 `#` 注释追加，不参与会话恢复；读取标题失败不影响核心功能。
6. 将结果包装成 Markdown 代码块，通过 PowerShell `Set-Clipboard` 写入剪贴板。

```shell
@echo off
setlocal DisableDelayedExpansion

rem 将 PowerShell 命令拆分为多个易读片段。
rem 会话 ID 必须直接来自环境变量，不能来自 pane 标题。
set "PS_COMMAND=$sessionId = $env:CLAUDE_CODE_SESSION_ID; "
set "PS_COMMAND=%PS_COMMAND%$resumeCommand = 'claude --resume'; "

rem 优先使用 Claude Code；只有 Claude ID 为空时才回退到 Codex。
set "PS_COMMAND=%PS_COMMAND%if ([string]::IsNullOrWhiteSpace($sessionId)) { "
set "PS_COMMAND=%PS_COMMAND%$sessionId = $env:CODEX_SESSION_ID; "
set "PS_COMMAND=%PS_COMMAND%$resumeCommand = 'codex resume' }; "

rem 如果没有有效的会话 ID，则停止生成恢复命令。
set "PS_COMMAND=%PS_COMMAND%if ([string]::IsNullOrWhiteSpace($sessionId)) { "
set "PS_COMMAND=%PS_COMMAND%Write-Error 'CLAUDE_CODE_SESSION_ID or CODEX_SESSION_ID not found'; "
set "PS_COMMAND=%PS_COMMAND%exit 1 }; "

rem 读取 tmux pane 标题，仅作为供人识别的可选元数据。
rem pane 标题绝不用于识别或恢复会话。
set "PS_COMMAND=%PS_COMMAND%$description = (wsl.exe -d ubuntu20 bash -lc 'source /home/ryan/.zshrc 2>/dev/null; tmux display-message -p ''#{pane_title}'' 2>/dev/null' | Select-Object -First 1); "

rem 清理终端转义序列和装饰性盲文字符。
set "PS_COMMAND=%PS_COMMAND%$description = ([string]$description).Trim() -replace '\x1b\[[0-9;]*[A-Za-z]',''; "
set "PS_COMMAND=%PS_COMMAND%$description = $description -replace '[\u2800-\u28FF]',''; "
set "PS_COMMAND=%PS_COMMAND%$description = $description.Trim(); "

rem 忽略无法描述当前任务的通用 shell 标题。
set "PS_COMMAND=%PS_COMMAND%if ($description -match '^(D:\|pwsh|init$|Administrator)') { $description = '' }; "

rem 组装恢复命令，并仅将标题作为注释追加到命令末尾。
set "PS_COMMAND=%PS_COMMAND%$resume = $resumeCommand + ' ' + $sessionId; "
set "PS_COMMAND=%PS_COMMAND%if ($description) { $resume += ' # ' + $description }; "

rem 将命令格式化为 c 代码块，复制到剪贴板，并输出以便核对。
set "PS_COMMAND=%PS_COMMAND%$newline = [Environment]::NewLine; "
set "PS_COMMAND=%PS_COMMAND%$fence = ([char]96).ToString() * 3; "
set "PS_COMMAND=%PS_COMMAND%$text = $fence + 'c' + $newline + $resume + $newline + $fence; "
set "PS_COMMAND=%PS_COMMAND%Set-Clipboard -Value $text; Write-Output $text"

rem 执行 PowerShell，并将其退出码返回给调用方。
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "%PS_COMMAND%"
exit /b %ERRORLEVEL%
```
