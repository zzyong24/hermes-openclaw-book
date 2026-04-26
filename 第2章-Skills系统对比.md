# 第二章：Skills 系统对比 — 谁真正解决了 Skill 管理的难题？

> 📌 **章节性质说明**
>
> 本章前半部分（5 个系统性问题）是月明的**原创分析**，来自源码验证和实战观察。
> 后半部分（OpenClaw Skills 实战配置）包含从 OpenClaw 文档和源码整理的内容。

---

## 2.1 Hermes Skills 的 5 个系统性问题

### 📖 源码证据：Skill 创建机制

> 以下来自 Hermes **源码分析**（`tools/skill_manager_tool.py`），**不是官方文档**：

```python
# tools/skill_manager_tool.py - SKILL_MANAGE_SCHEMA description
"""
Create when: complex task succeeded (5+ calls), errors overcome,
user-corrected approach worked, non-trivial workflow discovered,
or user asks you to remember a procedure.
"""
```

### 问题一：Skill 创建是 Nudge + 背景审查双重机制



**源码机制**（`run_agent.py:8843-8849, 11703-11709, 2796-2842`）：

`_iters_since_skill` 每 tool-call iteration 累加，达到 `_skill_nudge_interval`（默认 10）后触发**两个动作**：

1. **Nudge 提醒**：给主 Agent 发一条消息建议创建 Skill（需要显式调用 `skill_manage()`）
2. **背景审查（自动）**：响应发送后，在**后台线程**中fork一个新的 AIAgent，注入 `_SKILL_REVIEW_PROMPT`，这个审查 Agent **自主调用** `skill_manage()` 创建或更新 Skill，**无需主 Agent 确认，也无需用户确认**

Hermes 确实存在自动创建 Skill 的机制：后台审查线程会自主调用 `skill_manage()`，无需主 Agent 确认，也无需用户确认。

但「经验 ≠ 技能」的问题仍然存在：

- 如果 Agent 通过 `skill_manage()` 创建了一个 Skill，**没有任何验证机制**确认这个 Skill 是否真的有效
- Skill 一旦创建，**永久存活**（没有负熵机制）
- 临时调试产生的 Skill 和真正有用的 Skill 混在一起，无法区分

这对应到自然系统里，是"工作记忆直接当成程序性记忆"——没有巩固，没有筛选。

![自动创建 Skill 失控——老妈的"我吃过的盐比你吃过的米多"](assets/ch02-auto-creation.png)

> 💡 【3句话版本】
> - 它就像**老妈的"我吃过的盐比你吃过的米多"**——Agent 用了一次觉得好，就创建一个 Skill 说自己"会了"，但其实只是碰巧成功了一次。
> - 但问题是**Skill 一旦创建就永久存活**，没有任何验证机制确认它是否真的有效，临时调试产生的 Skill 和真正有用的 Skill 混在一起无法区分。
> - 解决办法是**关掉 `creation_nudge_interval`（设为 0 或者 较大的值）+ 每周 review 一次**，不用的 Skill 直接删。

---

### 📖 官方内容：Hermes SKILL.md 格式

> 以下来自 Hermes 官方文档对 SKILL.md 格式的描述：

```yaml
# SKILL.md 标准格式
---
name: skill-name
description: Skill 描述
triggers:
  - 触发关键词
  - 触发关键词
related_skills:
  - other-skill-name
---
# Skill 正文内容
```

> 🧠 **原创分析：related_skills 被解析和返回，但无任何逻辑消费它**
>
> 源码里 `related_skills` 确实被解析（`skills_tool.py:1187-1188`）并写入返回 JSON（`line 1281`），但没有任何代码引用这个字段做 Skill 组合或关联推理。它被解析了，只是没有被使用。

### 问题二：无负熵机制 — Skill 只增不减

**负熵（Negentropy）**：系统要维持稳定性，必须有"排出混乱"的机制。在自然界，这个机制叫"死亡"。

Hermes Skills 的问题是：**只能增，不能减。**

- 临时 Skill 被创建了 → 永久存活
- 过时 Skill 没人用 → 不会自动禁用
- 重复 Skill 出现了 → 不会提醒你

**这就像生小孩不养小孩——只管生，不管死。**

对比 OpenClaw：也没有自动删除机制，但至少 Skill 列表是显式的，你可以看到全貌。Hermes 的 Skill 是散落在文件系统里的，很多是 Agent 自己创建的，你可能根本不知道它存在。

![无负熵机制——生小孩不养小孩](assets/ch02-entropy.png)

> 💡 【3句话版本】
> - 它就像**办公室的垃圾桶满了没人倒**——Skill 一旦创建就永远不会消失，过时的、重复的、临时调试用的，全都堆在那儿。
> - 但问题是**你不知道该倒哪一只垃圾桶**——没有使用追踪，不知道哪些 Skill 真的被调用过，哪些从创建那天起就没人看过。
> - 解决办法是**定期 `find ~/.hermes/skills/ -name "SKILL.md" -mtime +30` 查修改时间**，长期未动的 Skill 优先考虑删除。

### 问题三：无关系机制 — related_skills 被解析但从未被消费

Hermes 的 SKILL.md 格式里有一个 `related_skills` 字段，设计意图是声明 Skill 之间的关系。

**源码实际情况**：`skills_tool.py:1187-1188` 确实解析了这个字段，`line 1281` 把它写入了返回 JSON。但没有任何代码引用这个字段做组合、推理或 Skill Selection。Skill 和 Skill 之间仍然是孤岛。

你想让两个 Skill 共享一段共同内容？做不到，只能复制粘贴。

![无关系机制——书架没有分类标签](assets/ch02-no-relation.png)

**实战影响：**
一个 "TDD 工作流" 的内容，可能同时存在于：
- `tdd-workflow` Skill
- `test-writing` Skill
- `subagent-testing` Skill

更新一次，得改三处。遗漏了某处，就出现了不一致。

> 💡 【3句话版本】
> - 它就像**图书馆的书架没有分类标签**——`related_skills` 字段被写进系统了，但系统根本不看它，书和书之间依然是孤岛。
> - 但问题是**你更新一本书的内容，另外两本同样讲 TDD 的书不会自动跟着更新**——三个地方三份内容，改一处漏两处。
> - 解决办法是**不要在多个 Skill 里写重复内容**，如果两个 Skill 描述有超过 50% 关键词重叠，直接合并。

### 问题四：无冲突检测 — 只查名字，不查语义

Hermes 创建 Skill 时，**只检测名字冲突**。

如果已经有一个叫 `excalidraw` 的 Skill，不能再创建第二个同名的。

但如果两个 Skill 做几乎一样的事，只是名字不同？**完全不会提醒你。**

这就是实际场景：4 个绘图 Skill 共存，功能重叠，Hermes 没有任何提醒。



> 💡 【3句话版本】
> - 它就像**健身房不记录你去了几次**——只知道卡里有钱，但不知道这张卡到底有没有被刷过、用了哪个器械。
> - 但问题是**你永远不知道该清理哪张卡**——`skill_view()` 被调用多少次没有记录，Skill 生态健康度无法评估。
> - 解决办法是**用文件修改时间代替使用追踪**（`ls -lt`），长期未修改的 Skill 大概率没被用过，优先 review。

![无冲突检测——健身房不记录去了几次](assets/ch02-no-conflict.png)

### 问题五：无使用追踪 — 没有任何调用数据

**这是最让人震惊的问题。**

Hermes 的源码里，**`skill_view()` 函数被调用了多少次，没有记录。**

- 哪个 Skill 最高频使用？不知道
- 哪个 Skill 从创建到现在一次都没加载过？不知道
- 你的 Skill 生态系统的健康度如何？无法评估

**这就像你健身房卡到期了，但你根本不知道自己去没去过。** 因为没有人记录。

> 💡 【3句话版本】
> - 它就像**健身房不记录你去了几次**——`skill_view()` 被调用多少次没有记录，你不知道哪张卡真的在被使用。
> - 但问题是**你不知道该停哪张卡**——没有使用数据，无法评估 Skill 生态系统的健康度，也不知道该清理哪些 Skill。
> - 解决办法是**用文件修改时间代替使用追踪**（`ls -lt ~/.hermes/skills/`），长期未修改的 Skill 优先 review 后删除。

![无使用追踪——健身房卡到期不知道去没去过](assets/ch02-no-usage-track.png)

---

## 2.2 🎯 类比：Skills 生态就像办公室里的打印机

办公室里有一台打印机，大家都在用，但没人维护：

- **问题一（经验 ≠ 技能）**：有人打印了一份合同，就觉得"我懂打印机了"，然后给打印机写了一份"使用指南"。但他其实只会打印合同，复印、打扫描都不会。

- **问题二（无负熵）**：打印机坏了，没人修，也没人说"这台该报废了"。它就这么坏着放在那儿，占地方，还有人继续往它那儿送纸。

- **问题三（无组合）**：打印机的说明书有 10 份，分散在各个部门。一份讲怎么装纸，一份讲怎么换墨，一份讲怎么复印——但它们之间没有任何关联，更新一份不代表更新另一份。

- **问题四（无冲突检测）**：市场部说"这台是 HP 的，联想的不行"。IT 部说"联想性价比高，HP 性价比低"。两套说法同时存在，打印出来的东西质量参差不齐，但没人知道该听谁的。

- **问题五（无使用追踪）**：打印机被用了多少次？哪个功能用得最多？哪个月保养过？**没人知道。** 就像 Hermes 的 `skill_view()` 一样，没人记录。

![Skills 生态——打印机类比](assets/ch02-skills-printer.png)

---

## 2.3 五个问题的互相强化：必然走向最大熵

```mermaid
flowchart TD
    A["问题一：Nudge 提醒 + 背景审查<br>Skill 质量无验证"] --> Z["系统走向<br>最大熵 Maximum Entropy"]
    B["无负熵机制<br>Skill 只增不减"] --> Z
    C["无关系机制<br>related_skills 被解析但无消费"] --> Z
    D["无冲突检测<br>互相踩的 Skills 共存"] --> Z
    E["无使用追踪<br>不知道该删哪一个"] --> Z

    E --> B
    E --> A
    B --> A
```

> **"只管生不管养，比不自动创建问题更严重。"** — 月明

---

## 2.4 OpenClaw 的 Skills 体系解决了什么？

### 📖 官方内容：OpenClaw Skills 架构

> 以下来自 OpenClaw 官方文档对 Skills 架构的描述：

```
OpenClaw Skills 系统设计原则：
1. <available_skills> 在 session 构建时通过 ensureSkillSnapshot() 生成
2. 之后是静态 XML，不再有任何阻塞
3. 所有 skill 读取都是按需的（Agent 主动用 read tool 读 SKILL.md）
4. snapshot 版本校验基于 skill 文件的 modification time + mtime 检查
```

> 🧠 **原创分析：解决了什么，没解决什么**
>
> **✅ 解决的问题：** 阻塞问题（一次性注入）、缓存失效机制（snapshot 版本校验）。
> **❌ 没解决的问题：** 无关系系统（扁平化注入）、无自动创建（Agent 发现好 workflow 只能告诉用户"帮我安装"）。

### 解决的问题

**✅ 问题一（阻塞）：存疑**

`skill_view()` 确实是同步函数（`skills_tool.py:823`，无 async/threading），但"卡住主 Agent loop"的说法无源码印证。Tool call 本身是 Agent loop 的正常步骤，不存在额外阻塞。OpenClaw 按需读取也并非因为 Hermes 的同步读取有问题，而是架构设计不同。

OpenClaw 的 `<available_skills>` 是**一次性注入**的——在 session 构建时通过 `ensureSkillSnapshot()` 生成一次，之后是静态 XML，不再有任何阻塞。所有 skill 读取都是按需的（Agent 主动用 `read` tool 读 SKILL.md）。

**✅ 问题二（无缓存）：解决了**

OpenClaw 有 snapshot 版本校验：`getSkillsSnapshotVersion()` 对 skill 目录树做哈希或 mtime 检查，匹配才复用，否则重建。比 Hermes 的被动失效更主动。

**✅ 问题四（无失效机制）：解决了**

`shouldRefreshSnapshot` 基于 skill 文件的 modification time + snapshot 版本对比，skill 更新后下一次回复前会触发重建。Hermes 依赖下次 session 启动才能刷新。

### 没解决的问题

**❌ 问题三（无关系系统）：没解决**

OpenClaw 完全没有 `related_skills` 这类 metadata。`<available_skills>` 把所有 skill 扁平化注入 system prompt，靠 description 模糊匹配。

**实战风险：两个 skill 如果 description 关键词重叠，Agent 可能选错 skill。**

比如同时有 "debugging" 和 "systematic-debugging"，description 都提到 "bug"，模型可能随机选一个。

**❌ 问题五（Agent 无法自主创建）：没解决**

OpenClaw 的 skill 安装依赖 `skillflag install` CLI，Agent 侧没有 skill 创建接口。Agent 发现了一个好的 workflow，只能告诉用户"帮我安装这个 skill"，无法自主创建。

---

## 2.5 OpenClaw vs Hermes Skills 机制对比

```mermaid
flowchart LR
    subgraph Hermes["Hermes Skills"]
        H1["skill_manage()<br>Nudge提醒 + 背景审查<br>（默认10次触发，自动创建）"]
        H2["skill_view()
同步执行
无异步优化"]
        H3["LRU dict<br>被动失效"]
        H4["related_skills<br>被解析返回<br>无逻辑消费"]
        H5["只检名字<br>不检语义"]
        H6["无使用追踪<br>不知道谁被调用"]
    end

    subgraph OpenClaw["OpenClaw Skills"]
        O1["CLI 手动安装<br>无自动创建"]
        O2["read tool<br>按需读取"]
        O3["snapshot 版本<br>主动校验"]
        O4["扁平化注入<br>无关系字段"]
        O5["无冲突检测<br>靠 description"]
        O6["无使用追踪<br>不知道谁被调用"]
    end

    H2 -.->|✅ 解决| O2
    H3 -.->|✅ 解决| O3
    H1 -.->|❌ 未解决| O1
    H4 -.->|❌ 未解决| O4
    H5 -.->|❌ 未解决| O5
    H6 -.->|❌ 未解决| O6
```

---

## 2.6 实战建议：两套系统各在什么场景下表现最好？

### Hermes Skills 适合的场景

| 场景 | 原因 |
|------|------|
| **你想让 Agent 自主学习工作流** | Hermes 支持 Agent 自主创建 Skill |
| **你需要接入外部 Memory Provider** | Hermes 有 MemoryProvider 插件架构 |
| **你有多消息平台的需求** | Hermes 支持 18 个消息平台 |
| **你愿意投入时间手动维护** | Hermes 的 Skill 需要定期人工审核清理 |

### OpenClaw Skills 适合的场景

| 场景 | 原因 |
|------|------|
| **你希望 Skills 稳定、不乱增长** | OpenClaw 没有自动创建机制 |
| **你需要语义搜索能力** | OpenClaw 有 LanceDB + QMD，向量搜索 |
| **你需要飞书深度集成** | OpenClaw 的飞书适配更深度 |
| **你想减少维护负担** | Skills 列表显式，snapshot 机制防止污染 |

### 共用的最佳实践

**① 给每个 Skill 写清晰的 description**

description 是 Agent 选 Skill 的主要依据。写得模糊会导致选错 Skill。

**② 给 Skill 按功能分类目录**

**③ 不要创建功能重叠的 Skill**

如果两个 Skill 的 description 有超过 50% 的关键词重叠，合并它们。

---

## 2.7 小结

| 维度 | Hermes Skills | OpenClaw Skills |
|------|---------------|-----------------|
| **创建机制** | ✅ 自动（Nudge + 背景审查，默认10次） | ❌ 无 |
| **阻塞问题** | 同步执行（存疑） | ✅ 一次性注入 |
| **缓存机制** | 两层（被动） | Snapshot 版本校验（主动） |
| **失效刷新** | 下次 session 启动 | 下次回复前自动 |
| **关系系统** | `related_skills`（被解析但无消费） | 无 |
| **冲突检测** | 只检测名字 | 无 |
| **使用追踪** | ❌ 完全没有 | ❌ 完全没有 |
| **维护负担** | 高（Skill 泛滥） | 低（显式控制） |

**核心结论：Hermes 的自动创建设计比 OpenClaw 更激进——后台审查线程会自主调用 `skill_manage()`，这解释了为何 Skill 会不断积累。OpenClaw 显式安装虽然保守，但避免了Skill 污染。主要差距在于 Hermes 缺少负熵机制和使用追踪。**

如果你用 Hermes，**主动维护是必须的**。如果不想花时间维护，迁移到 OpenClaw 是更务实的选择。

---

## 2.8 Skills 实战配置 — 可操作的指南

### 怎么关闭 Hermes 自动创建 Skill？

Hermes 的自动创建由两套机制驱动，关闭方式如下：

```yaml
# ~/.hermes/config.yaml
skills:
  creation_nudge_interval: 0  # 关闭 Nudge 提醒和背景审查触发（设为0即完全关闭）
```

> ⚠️ **注意**：配置路径是 `skills.creation_nudge_interval`（在顶层 `skills` 下），**不是** `agent.skill_manager.disabled`（该结构不存在）。`creation_nudge_interval` 设为 0 时，Nudge 提醒和背景审查触发都会被禁用，Skill 创建完全依赖 Agent 显式调用 `skill_manage()`。

验证方式：
```bash
grep -r "creation_nudge_interval" ~/.hermes/config.yaml 2>/dev/null || echo "未配置，使用默认值（10次触发）"
```
### 怎么定期 review Skill？（脚本 + 命令）

**方法一：手动 audit（适合每月一次）**

```bash
# 环境：macOS / Linux，Hermes 运行中

# 1. 列出所有 skills（含创建时间）
echo "=== Hermes Bundled Skills ==="
ls -lt ~/.hermes/skills/hermes/

echo "=== User-Created Skills ==="
ls -lt ~/.hermes/skills/

# 2. 查看 darwin-skill（达尔文优化入口）
# 详见：https://github.com/alchaincyf/darwin-skill
# 安装方式：openclaw skillflag install --scope user https://github.com/alchaincyf/darwin-skill

# 3. 查看哪些 skills 从未被调用（以文件修改时间判断）
find ~/.hermes/skills/ -name "SKILL.md" -type f \
  -not -path "*/hermes/*" \
  -exec stat -f "%m %N" {} \; \
  | sort -n | head -10
```

**方法二：用 darwin-skill 自主优化**

> 📌 **工具来源**：darwin-skill 开源项目 — https://github.com/alchaincyf/darwin-skill

```bash
# 在 Hermes 里说："帮我用达尔文方法 review 所有 skills"
# 触发 darwin-skill，按 8 个维度对每个 skill 评分：
#
# 1. Structure（结构完整性）：SKILL.md frontmatter 是否规范
# 2. Clarity（描述清晰度）：description/triggers 是否无歧义
# 3. Coverage（触发覆盖）：triggers 是否覆盖真实使用场景
# 4. Recency（时效性）：内容是否与当前系统版本匹配
# 5. Compatibility（兼容性）：是否依赖已废弃的 tool/API
# 6. Uniqueness（唯一性）：是否与现有 skill 功能重复
# 7. Utility（实用性）：实际被调用的频率
# 8. Maintainability（可维护性）：更新一次能覆盖多少场景

# 达尔文优化的工作流：
# 1. 评估：8维度评分
# 2. 改进：对评分 < 7 的 skill 进行自动修改
# 3. 实测：用 test prompts 验证改进效果
# 4. 人类确认：修改后等待用户确认
# 5. Git 版本控制：每次改进都 git commit，支持回滚
```

### 飞书多账号 Bot 配置经验

实际踩过的坑：

**坑 1：多 Bot 共享同一个 skills 目录导致 skill 冲突**

```yaml
# 错误配置：两个 bot 用同一个 workspace
# ~/.openclaw/config.yaml
agents:
  bot_a:
    workspace: ~/.openclaw/workspace/main  # ❌ 与 bot_b 共享
  bot_b:
    workspace: ~/.openclaw/workspace/main  # ❌ 冲突

# 正确配置：每个 bot 独立 workspace
agents:
  bot_a:
    workspace: ~/.openclaw/workspace/bot_a
  bot_b:
    workspace: ~/.openclaw/workspace/bot_b
```

**坑 2：飞书多账号的 session key 路由错误**

```yaml
# ~/.openclaw/config.yaml
feishu:
  session_scope: group_sender  # 推荐：一个群+一个发送者 = 一个 session
  # ❌ 不要用 group（一个群一个session会导致多用户消息混在一起）
  # ⚠️ group_topic 只在话题群生效，普通群会退化为 group
```

**坑 3：Skills 在多 bot 环境下重复安装**

```bash
# 在 bot_a 的 workspace 安装 skill
openclaw skillflag install --scope user --target bot_a \
  https://github.com/example/skill

# ⚠️ 错误：在 bot_b 的 workspace 再次安装同一个 skill
# 结果：同一个 skill 被安装两次，占用空间，且 snapshot 版本可能不一致

# 正确做法：在 global scope 安装一次，所有 bot 共享
openclaw skillflag install --scope global \
  https://github.com/example/skill
```

**飞书 Bot Skills 配置 checklist**

```
✅ 每个 bot 有独立 workspace（避免 skills 互相污染）
✅ 高频复用的 skill 在 global scope 安装
✅ Bot-specific skill 在对应 workspace 安装
✅ Skills 安装后重启 gateway 使其生效
✅ 定期检查 skills 目录：`ls ~/.openclaw/workspace/{bot}/skills/`
✅ 确认 skills 的 scope 层级：global > user > repo
```

---

## 📦 SKILL：第二章实战精华

- [Hermes 实战指南（All-in-One）](Hermes配置与优化.md)
- [OpenClaw 实战指南（All-in-One）](OpenClaw配置与优化.md)

