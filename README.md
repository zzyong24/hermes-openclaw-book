# Hermes + OpenClaw 对比电子书

> 实战者的双 Agent 架构手册——从踩坑经历到选型决策

---

## 本书目标

深入对比 Hermes 和 OpenClaw 两大 AI Agent 框架在 **Skills、SubAgent、Memory** 三大核心系统上的设计差异和实战表现。

---

## 章节导航

### [第一章：Hermes 的三大核心问题](./ch01-hermes-memory-disorder.md)

从月明的真实经历（两周使用）出发，概览 Hermes 的三个核心问题：
- Skills 系统灾难（多 Skill 互相踩、越积越多没人管）
- 记忆紊乱（旧事重现、新记忆不生效）
- SubAgent 任务中断

> 📌 **章节特点**：引子，类比丰富（花园/土壤/浇水装置）

---

### [第二章：Skills 系统对比](./ch02-skills-comparison.md)

Hermes Skills 的 5 个系统性问题 + OpenClaw Skills 体系分析：
- 经验 ≠ 技能（门槛过低）
- 无负熵机制（只增不减）
- 无组合机制（孤岛效应）
- 无冲突检测（只查名字）
- 无使用追踪（黑盒子）

> 📌 **章节特点**：原创分析，类比（"办公室打印机"），附 SKILL 格式实战干货

---

### [第三章：SubAgent 机制对比](./ch03-subagent-comparison.md)

分析 Hermes 的 SubAgent 机制：
- **claude-code 和 codex 是内置 tool，不是 skill**（在 `tools/` 目录下）
- 真正的问题是：外部工具调用，而不是真正的 Session 隔离委托
- OpenClaw sessions_spawn + ACP 的完整 spawn 链
- Child 超时不自杀、Stream relay 数据丢失等坑位

> 📌 **章节特点**：流程图解（Chain/Fan-out/Tree 三种编排模式）

---

### [第四章：Memory 系统对比](./ch04-memory-comparison.md)

基于 Hermes 源码分析，揭示 Memory 紊乱的真正根因：
- **MEMORY.md 超载**：2996 chars > 2200 限制
- **flush_memories 误写**：Context Compression 在 90 轮上下文压力下把旧信息写入 memory
- **nudge_interval=10**：Background Review 每 10 轮触发，累积效应显著
- **三层次上下文结构**：系统级上下文、Session Context、对话历史
- Hermes：frozen snapshot 不同步、Provider 拼接问题
- OpenClaw：QMD CLI 冷启动、memory.db 并发 lock

> 📌 **章节特点**：源码级分析，架构图 + 对比表

---

### [第五章：选型建议与避坑指南](./ch05-selection-guide.md)

实战者的决策手册：
- 快速选型决策树
- Hermes/OpenClaw 各适用场景
- 完整的避坑速查表

> 📌 **章节特点**：决策流程图、各系统避坑清单

---

### [第六章：月明的实践](./ch06-my-practice.md)

月明的多虾架构落地经验：
- OpenClaw 多 Agent 体系（猪猪虾 + 5 只虾）
- 飞书多 Bot 账号配置
- Hermes Gateway 安装（WebSocket 模式）
- 踩坑总结
- 飞书 Bot 权限配置

> 📌 **章节特点**：实战经验，可操作的配置指南

---

## SKILLS 目录

独立的 SKILL 文件目录，可被 AI Agent 直接加载运行：

```
SKILLS/
├── HERMES/           # 给 Hermes 用的 skill
│   ├── hermes-skills-auto-creation-control.SKILL.md   # 控制 skills 自动创建
│   └── hermes-subagent-analysis.SKILL.md             # SubAgent 机制分析
└── OPENCLAW/         # 给 OpenClaw 用的 skill
    ├── openclaw-memory-concurrency-guide.SKILL.md    # Memory 并发避坑
    └── openclaw-sessions-spawn-guide.SKILL.md       # ACP spawn 实战指南
```

主文档只引用 SKILL 文件，不重复内容。

---

## 特色说明

### 📊 丰富的图表

每章包含 2-3 个 Mermaid 图：
- 架构图（Memory、Skills、SubAgent）
- 流程图（因果链、spawn 链）
- 对比表（Hermes vs OpenClaw）

### 🎯 生活化类比

技术概念用通俗故事表达：
- Skills 生态 → "办公室打印机大家用但没人维护"
- Skills 只增不减 → "健身房卡到期了但不知道自己去没去过"
- Hermes vs OpenClaw → "花园 vs 宜家家具"

### 📦 SKILL 格式干货

每章的"避坑清单/实战建议"改成 SKILL.md 格式：
- YAML frontmatter + name + description + triggers + tags
- 可以被 AI Agent 直接调用

---

## 来源说明

本书综合了：
- **Hermes 源码分析**（MemoryManager、MemoryProvider 等）
- **OpenClaw 源码分析**（ACP spawn 链、Memory 系统等）
- **月明的真实实战经历**和踩坑总结
- **源码验证**：所有关键说法标注了源码位置

---

*最后更新：2026-04-25*
