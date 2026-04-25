---
name: hermes-skills-auto-creation-control
description: 控制 Hermes skills 自动创建行为 — 配置文件路径、关闭方法、验证命令
platform: hermes
triggers:
  - "Hermes skill 泛滥"
  - "skills 自动创建"
  - "skill 目录太乱"
  - "关闭 Hermes 自动创建"
tags:
  - hermes
  - skills
  - auto-creation
---

# 使用方法

## 根因说明

Hermes 有 skills 自动创建机制（auto-creation），当检测到重复任务模式时会自动生成新的 skill。这些 skill 存放在用户目录（`~/.hermes/skills/` 下的非官方目录），不是 Hermes 官方提供的。

## 控制方法

### 1. 查看当前状态

```bash
# 查看有多少 skills 是用户创建的（非 hermes 官方）
find ~/.hermes/skills/ -name "SKILL.md" -type f \
  -not -path "*/hermes/*" \
  -not -path "*/mcp/*" | wc -l

# 查看 Hermes 官方 bundled skills
ls ~/.hermes/skills/hermes/

# 查看当前 auto-creation 配置
grep -r "creation_nudge_interval\|auto_create_after" \
  ~/.hermes/config.yaml 2>/dev/null \
  || echo "未配置，使用默认值（10次 tool calls 触发）"
```

### 2. 关闭 auto-creation

```yaml
# ~/.hermes/config.yaml
agent:
  skill_manager:
    creation_nudge_interval: 0  # 完全关闭自动创建
    disabled:
      - auto_create_after_tool_calls
      - creation_nudge_interval
      - skill_auto_create
```

### 3. 定期 review 用户创建的 skills

```bash
# 查看用户创建的 skills（按修改时间）
find ~/.hermes/skills/ -mindepth 1 -maxdepth 1 \
  -not -name "hermes" -not -name "mcp" -type d

# 查看哪些 skills 从未被调用（以文件修改时间判断）
find ~/.hermes/skills/ -name "SKILL.md" -type f \
  -not -path "*/hermes/*" \
  -exec stat -f "%m %N" {} \; \
  | sort -n | head -10
```

### 4. 推荐的 skills 目录结构

```bash
~/.hermes/skills/
├── hermes/          # Hermes 官方（只读，不要改）
├── mcp/             # MCP 协议集成（只读）
└── [user-created]/  # 用户创建的 skills（定期 review）
```

**不要删除 hermes/ 和 mcp/ 下的任何内容。**

## 达尔文 skill 优化

> 📌 **工具来源**：darwin-skill 开源项目 — https://github.com/alchaincyf/darwin-skill

可以用达尔文方法对所有 skills 做 8 维度评分，定期淘汰低质量 skills。
