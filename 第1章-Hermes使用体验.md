# 第一章：Hermes 的三大核心问题——我的踩坑总结

> 📌 **章节性质说明**
>
> 本章是全书的**总起**，基于月明两周 Hermes 使用经历的**真实踩坑**。
> 每个问题会在后续章节详细展开。

---

## 1.1 先说我的经历

我用了两周 Hermes。

两周时间里，Hermes 帮我处理了大量工作——写文章、做调研、写代码、回答问题。效率确实高，我越用越依赖它。

但用的时间越长，问题也开始暴露出来。

**问题开始出现的时候，是在我让连续做了几个复杂任务之后。**

---

## 1.2 问题是怎么发现的

### 第一天：画图风格不一致

第一天我让 Hermes 给我生一批配图，总共 8 张。

我预期的结果是一套风格统一的图——就像同一个设计师的作品。但实际出来的 8 张图，让我怀疑这不是同一个人画的：

- 图1：扁平风格，蓝色主调
- 图2：写实风格，灰色调
- 图3：手绘风格，线条粗犷
- 图4-8：各有各的问题



我的第一反应是：工具出问题了？模型不统一？

检查了一圈，发现模型是同一个。

问题不在模型，在 Hermes 本身。

### 第三天：Skill 越积越多

用到第三天，我开始注意到一个问题——Skills 目录越来越大了。



Hermes 会根据我的使用情况自动创建 Skill。有些是我让它记住的流程，有些是它自己判断"这个任务模式值得保留"而创建的。

我打开 Skills 目录一看，发现已经有几十个 Skill 了，其中绘图相关的就有 4 个。

这 4 个绘图 Skill 什么关系？互相踩。同样一个"画一张图"的请求，每次选到的 Skill 不同，结果风格完全不同。

### 第五天：记忆开始混乱

这是最让我头疼的。



新建的记忆，下一个问题里就没了。但几天前被否定的方案，突然又冒出来。

我明明说过"以后这种图用扁平化风格"，下一个问题它又选了另一种风格。

更奇怪的是，有时候它会"记起"一些我从来没说过的内容——或者是我说过、但后来明确否定了的内容。

---

## 1.3 三个核心问题

两周时间，我踩出了三个大类的问题：

```mermaid
graph LR
    A[画图风格不一致] --> P1[Skills 系统问题]
    B[Skill 越积越多] --> P1
    C[记忆混乱] --> P2[记忆系统问题]
    D[新建记忆不生效] --> P2
    E[旧事重现] --> P2
    F[任务中断] --> P3[SubAgent 问题]
```

### 问题一：Skills 系统灾难

**表现：** 多 Skill 互相踩、越积越多没人管

4 个绘图 Skill 功能重叠，Hermes 选 Skill 的逻辑是随机的——看关键词、看近期提及、看心情，不是看哪个最适合。

更严重的是，没有使用追踪。我不知道哪些 Skill 真的被用过、哪些从没触发过、哪些已经过时了。

> 💡 【3句话版本】
> - 它就像**打印机旁堆了一堆使用指南，但没人告诉你哪份是最新的**——4 个绘图 Skill 共存，Hermes 随机选一个，结果每次画的风格都不一样。
> - 但问题是**没人知道该丢掉哪份**——没有使用追踪，不知道哪份指南真的被用过，哪份从第一天就躺在那儿积灰。
> - 解决办法是**关掉自动创建（`creation_nudge_interval: 0`）+ 每周手动 review 一次**，把不用的 Skill 直接删掉。

### 问题二：记忆紊乱

**表现：** 新记忆不生效，旧事莫名重现

![记忆紊乱——两个秘书各记各的](assets/ch01-memory-chaos.png)

新建的记忆在下一个问题里消失了，但几天前被否定的方案又冒出来了。

后来 Hermes 自己诊断，发现是 MEMORY.md 超出限制，加上 Context Compression 机制在长会话时误写，导致旧信息被固化。

> 💡 【3句话版本】
> - 它就像**寄信后对方说没收到**——你确实发了（`add()` 写入了），但信卡在邮局里（`_system_prompt_snapshot` 冻结），下一轮对话看不到新内容。
> - 但问题是**旧信还在反复投送**——几天前被否决的方案被当成有效信息继续用，因为 snapshot 冻结后旧记忆没被正确清除。
> - 解决办法是**每次改完 MEMORY.md 开一个新 session**，或者把高频事实直接写死进 SOUL.md（绕过 memory 系统）。

### 问题三：SubAgent 任务中断

**表现：** 任务跑到一半断了，返回值不完整

用 Hermes 的 delegate_task 执行多步任务，结果任务跑到一半被打断，未完成的子 agent 返回 `summary: None`。

这不是因为"外部工具调用没有 session 隔离"——delegate_task 有完整的子 Agent 隔离机制（独立 conversation、独立工具集、parent_session_id 追踪）。

真正的原因是 **interrupt 传播机制**：当你在 subagent 执行期间发来新消息，Gateway 会调用 `parent.interrupt()`，这个调用会：
1. 设置 parent 的 `_interrupt_requested = True`
2. 传播 interrupt 信号给所有 `_active_children`（子 agent）
3. delegate_tool 检测到 parent 被 interrupt 后，**立即放弃未完成的子 agent**（状态标记为 `interrupted`，`summary: None`）

也就是说，subagent 跑了一半被打断，但它的中间进度全部丢失——不是排队等恢复，是直接丢弃。

> 💡 【3句话版本】
> - 它就像**派孩子去超市买东西，走到一半你喊他回来，但他回来时手里是空的**——你不知道他有没有买到，也不知道买到了什么。
> - 但问题是**水管里的水直接倒掉了**——interrupt 传播时，subtask 的中间进度全部丢弃，不是暂停等恢复，是直接清零。
> - 解决办法是**用 OpenClaw ACP `sessions_spawn`** 代替 Hermes `claude-code`/`delegate_task`，因为 ACP 有完整的 session 隔离和 stream relay。

---

## 1.4 三个问题的概览

| 问题 | 表现 | 根因 |
|------|------|------|
| Skills 灾难 | 多 Skill 互相踩、越积越多没人管 | 无负熵机制、无使用追踪 |
| 记忆紊乱 | 旧事重现、新记忆不生效 | MEMORY.md 超载 + Context Compression 误写 |
| SubAgent 中断 | 任务跑到一半断了，返回值不完整 | interrupt 来了直接丢弃未完成的子 agent 结果 |

---

## 1.5 类比

```mermaid
graph TB
    subgraph 问题["三个系统性问题"]
        S[Skills<br/>只管生不管养]
        M[Memory<br/>只管写不管清]
        A[SubAgent<br/>只管跑不管停]
    end
    
    subgraph 表现["表现"]
        S1[越积越多<br>不知道删哪个]
        M1[超载了还在写<br>旧事反复出现]
        A1[任务跑到一半<br>突然断了]
    end
    
    subgraph 类比["自然类比"]
        B[花园没人修剪]
        C[土壤越来越贫瘠]
        D[浇水到一半停掉]
    end
    
    S --> S1
    M --> M1
    A --> A1
    
    S1 -.-> B
    M1 -.-> C
    A1 -.-> D
```

- **Skills**：花园里自己长出来的植物，没有园丁修剪
- **Memory**：花园里的土壤，越用越贫瘠但没施肥清理
- **SubAgent**：花园里的自动浇水装置，浇到一半会突然停——而且水管里的水直接倒掉了，不会保留一滴

---

## 1.6 详细分析在后面

本章是引子，后面有详细分析：

- **第二章**：Skills 系统深度分析 + 解决方案
- **第三章**：SubAgent 机制分析 + OpenClaw ACP 对比
- **第四章**：Memory 系统深度分析 + OpenClaw Memory 对比

---

## 1.7 解决方案概览

| 问题 | 解决方案 |
|------|---------|
| Skills 灾难 | 关闭 auto-creation + 定期 review + 达尔文技能优化 |
| 记忆紊乱 | 调高 nudge_interval + 清理 MEMORY.md + 定期检查 |
| SubAgent 中断 | 用 OpenClaw ACP sessions_spawn 代替 Hermes claude-code |

---

## 1.8 自检命令

```bash
# 检查 MEMORY.md 大小
wc -c ~/.hermes/memories/MEMORY.md

# 检查 Skills 数量
find ~/.hermes/skills/ -name "SKILL.md" | wc -l

# 检查当前 memory 配置
grep -A5 "memory:" ~/.hermes/config.yaml
```
