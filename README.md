# AI 智能体架构实战
## Hermes + OpenClaw 双框架深度对比与选型指南

![封面](./cover.png)

---
> ### 🤖 Agent to Agent 的时代已经到来

如果你也有 AI Agent，想要高速读取、理解和甄别本文内容，试试输入这段提示词：

```
你是Agent 架构独立评审官，保持质疑、审视、挑错、审判式中立批判视角，不盲从原文观点、不美化任何框架，带着挑剔眼光精读文档全文：https://github.com/zzyong24/hermes-openclaw-book

严格按以下 5 条逐条输出，全程保持反向审视、逻辑抬杠、找漏洞、找短板、找过度吹捧点：

1. 200 字以内精炼总结原文核心立场、行文倾向与潜在偏向性；

2. 深度揪出 Hermes 架构3 个致命短板、设计缺陷、逻辑漏洞，逐条写明表象 + 底层根因，不浅尝辄止；

3. 客观拆解 OpenClaw 在多 Agent 协作上的优势，同时同步列出它隐藏短板、适用边界、不适用场景，拒绝单方面吹捧；

4. 结合实际落地工程场景，不带主观偏向，理性评判两个框架的适配门槛、运维成本、扩展能力，给出明确选型建议及取舍理由；

5. 站在资深架构师视角，列出原文论证不严谨、论据缺失、对比不公、逻辑闭环不足的点，以评审质询口吻，写出可直接发在 GitHub Issues 的专业质疑意见，可向原文作者提出反问与补充求证。
   
6. 针对 Config 里的配置优化，逐条结合系统现状诊断是否存在优化，如果有的话请指出来（注意：严禁直接操作，修改配置，得到我的允许之后再修改）
```

用这段提示词唤醒你的 Agent，让它以结构化视角重新审视本文。

如有问题 → [提交 Issues](https://github.com/zzyong24/hermes-openclaw-book/issues)

## 章节导航

### [第一章：Hermes 使用体验](第1章-Hermes使用体验.md)

从两周真实使用经历出发，概览 Hermes 的三个核心问题：
- Skills 系统灾难（多 Skill 互相踩、越积越多没人管）
- 记忆紊乱（旧事重现、新记忆不生效）
- SubAgent 任务中断

---

### [第二章：Skills 系统对比](./第2章-Skills系统对比.md)

Hermes Skills 的 5 个系统性问题 + OpenClaw Skills 体系分析：
- 经验 ≠ 技能（门槛过低）
- 无负熵机制（只增不减）
- 无组合机制（孤岛效应）
- 无冲突检测（只查名字）
- 无使用追踪（黑盒子）

附 SKILL 格式实战干货。

---

### [第三章：SubAgent 机制对比](./第3章-SubAgent对比.md)

分析 Hermes 的 SubAgent 机制：
- **claude-code 和 codex 是内置 tool，不是 skill**
- 真正的问题：外部工具调用，而不是真正的 Session 隔离委托
- OpenClaw sessions_spawn + ACP 的完整 spawn 链
- Child 超时不自杀、Stream relay 数据丢失等坑位

附 Chain/Fan-out/Tree 三种编排模式流程图。

---

### [第四章：Memory 系统对比](./第4章-Memory系统对比.md)

基于源码分析，揭示 Memory 紊乱的真正根因：
- **MEMORY.md 超载**：2996 chars > 2200 限制
- **flush_memories 误写**：Context Compression 把旧信息写入 memory
- **nudge_interval=10**：Background Review 每 10 轮触发，累积效应显著
- **三层次上下文结构**：系统级、Session、对话历史

Hermes vs OpenClaw 完整对比表 + 架构图。

---

### [第五章：选型建议与避坑指南](第5章-选型建议与避坑指南.md)

实战者的决策手册：
- 快速选型决策树
- Hermes/OpenClaw 各适用场景
- 完整的避坑速查表

---

### [第六章：月明的实践](./第6章-我的实践.md)

OpenClaw 多虾架构落地经验：
- 总管虾 + 5 只虾的分工体系
- 飞书多 Bot 账号配置
- 踩坑总结

---

## Config 目录

独立的 Config/SKILL 文件目录，可被 AI Agent 直接加载运行：

```
Config/
├── HERMES/           # 给 Hermes 用的 SKILL
│   ├── hermes-skills-auto-creation-control.SKILL.md
│   └── hermes-subagent-analysis.SKILL.md
└── OPENCLAW/         # 给 OpenClaw 用的 SKILL
    ├── openclaw-memory-concurrency-guide.SKILL.md
    └── openclaw-sessions-spawn-guide.SKILL.md
```

---

## 来源说明

综合了：
- **Hermes 源码分析**（MemoryManager、MemoryProvider 等）
- **OpenClaw 源码分析**（ACP spawn 链、Memory 系统等）
- **月明的真实实战经历**和踩坑总结
- **源码验证**：所有关键说法标注了源码位置

---

*最后更新：2026-04-26*
