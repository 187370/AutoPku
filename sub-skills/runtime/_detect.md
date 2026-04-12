---
name: autopku-runtime-detect
description: 检测当前运行的 AI 环境（Claude Code / Codex / Other）
---

# AI 环境检测

## 检测方法

### Claude Code
```python
import os

is_claude_code = (
    os.environ.get("CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS") is not None or
    os.environ.get("CLAUDE_CODE") == "1"
)
```

特征：
- 环境变量 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`（启用 agent team）
- 支持 `Agent()` tool 创建并行 agent
- 支持 `SendMessage()` 进行 agent 间通信

### Codex
```python
import os

is_codex = os.environ.get("CODEX") == "1"
```

特征：
- 环境变量 `CODEX=1`
- 支持 native subagents
- 使用不同的 subagent API

### 其他 AI
无法识别时，回退到串行执行模式。

## 使用示例

```python
# 在 skill 中检测环境
import os

if os.environ.get("CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS"):
    runtime = "claude"
elif os.environ.get("CODEX") == "1":
    runtime = "codex"
else:
    runtime = "fallback"

# 根据环境选择执行方式
if runtime == "claude":
    # 使用 Agent() tool
    pass
elif runtime == "codex":
    # 使用 Codex subagent
    pass
else:
    # 串行执行
    pass
```
