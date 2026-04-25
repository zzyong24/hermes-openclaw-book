---
name: hermes-subagent-analysis
description: Hermes SubAgent 机制分析 — profile 机制、claude-code/codex 内置 tool 与 skill 的区别、外部工具调用与 session 隔离委托的本质差异
platform: hermes
triggers:
  - "Hermes SubAgent 机制"
  - "Hermes 有没有 subagent"
  - "claude-code skill"
  - "Hermes profile"
  - "subagent session 隔离"
tags:
  - hermes
  - subagent
  - profile
  - claude-code
---

# 使用方法

## Hermes 的 Profile 机制

Hermes 有 profile 机制，可以通过不同 profile 实现不同角色的 Agent。每个 profile 有独立的配置、skills 和行为模式。

## Hermes SubAgent 的真实机制

Hermes 的"subagent"是通过以下方式实现的：

### 1. Profile 切换（真正的多角色隔离）

```bash
# Hermes 通过切换 profile 实现不同角色
# ~/.hermes/profiles/
# ├── main/          # 主 Agent
# ├── coder/         # 编程 Agent
# └── creative/      # 创意 Agent

# 切换 profile
/hermes switch-profile coder
```

### 2. claude-code / codex skill（外部工具调用）

**重要：** `claude-code` 和 `codex` 是 Hermes **内置的 tool 实现**（在 `tools/` 目录下），不是 `~/.hermes/skills/` 里的 SKILL.md 文件。

```python
# Hermes Agent 源码
hermes-agent/
└── tools/
    ├── claude_code_tool.py   # 内置 tool
    └── codex_tool.py          # 内置 tool
```

它们的使用方式是 `tool_call`，调用外部 CLI 进程：
- 没有 session key
- 没有流式转发（streaming relay）
- 没有生命周期管理

### 3. 真正的问题

**这些是"外部工具调用"而不是"真正的 Session 隔离委托"。**

```
Hermes 的 subagent = 工具级委托（tool_call → 外部 CLI）
OpenClaw 的 subagent = Session 级隔离（sessions_spawn → ACP protocol）
```

## Hermes Skills 目录结构（必须分清）

```bash
~/.hermes/skills/
├── hermes/          # Hermes 官方/Bundled Skills
│   ├── skills-diagnosis/
│   ├── skills-judgment/
│   ├── hermes-approval-debugging/
│   └── ...（共10个官方 skill）
│
├── mcp/             # MCP 协议集成
│   ├── mcporter/
│   └── native-mcp/
│
└── [user-created]/   # 用户创建的 Skills（auto-creation 产物）
    └── ...（数量因使用情况而异）
```

**关键区别：**
- `claude-code` 和 `codex` 不在 `~/.hermes/skills/` 里
- 它们是内置 tool，在 `tools/` 目录下
- 所以"claude-code skill"这个说法其实不准确，应该叫"claude-code tool"

## 与 OpenClaw 的本质差异

| 维度 | Hermes | OpenClaw |
|------|--------|----------|
| SubAgent 机制 | Profile 切换 + tool_call | sessions_spawn + ACP |
| 隔离方式 | 无 session 隔离 | 独立 session key |
| 流式转发 | 无 | ✅ stream relay |
| 生命周期管理 | 无 | ✅ AcpSessionManager |
| 外部 CLI 调用 | ✅ claude-code/codex | ❌（格式不兼容） |

## 什么时候用 Hermes 的 claude-code？

- 需要调用本地 Claude Code CLI
- 纯文本任务

## 什么时候用 OpenClaw sessions_spawn？

- 需要实时看到子任务输出
- 需要 Parent-Child Session 隔离
- 飞书消息触发的多任务