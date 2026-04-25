# 第五章：选型建议与避坑指南 — 实战者的决策手册

> 📌 **章节性质说明**
>
> 本章基于前四章的结论汇总，是月明的**原创综合分析**。

---

## 5.1 一句话选哪个

**你想让 Agent 帮你干活，还是只是想有个省心的助手？**

```
想省心 → Hermes（注意几个坑，定期维护）
想深度定制、让 Agent 真正帮你干活 → OpenClaw
```

---

## 5.2 两种选择的核心区别

|          | Hermes           | OpenClaw       |
| -------- | ---------------- | -------------- |
| **核心理念** | 让 Agent 自己学，你来维护 | 你定制规则，Agent 执行 |
| **上手难度** | 低，用几次就开始"懂你"     | 高，要自己配置        |
| **维护成本** | 中等，要定期清理 Skills  | 低              |
| **稳定性**  | 偶尔会"失忆"          | 比较稳定           |
| **定制化**  | 有限，Agent 自己做主    | 很高，你说了算        |

**简单说：**

- Hermes 像请了个**聪明的实习生**——学东西快，但有时候会忘事
- OpenClaw 像**工具**——用起来要花时间 setup，但用好了很靠谱

---

## 5.3 什么时候选 Hermes？

### 适合用 Hermes 的场景

**场景一：想省心，不想花时间配置**

Hermes 用几次就开始"懂你"了。自动创建 Skill 功能会让它越来越顺手。

**前提：你要愿意定期维护（清理 Skill、注意那几个坑）。**

**场景二：多平台消息聚合**

Slack、Discord、Telegram 都要接？Hermes 支持 18 个平台，一个 Agent 全搞定。

**场景三：不想折腾，直接上手用**

不需要配置什么，接入飞书就能用。适合不想花时间折腾的人。

---

## 5.4 什么时候选 OpenClaw？

### 适合用 OpenClaw 的场景

**场景一：想让 Agent 真正帮你干活**

Hermes 帮你处理日常问答可以，但让它独立完成项目？差点意思。OpenClaw 的 subagent 机制更完整。

**场景二：飞书多账号 Bot 系统**

需要同时跑多个 Bot，每个 Bot 有独立角色？OpenClaw 的多 Bot 配置更成熟。

**场景三：深度定制，不接受"差不多"**

Hermes 会自己判断怎么做，但你有时候想要它按你的方式做。OpenClaw 你说了算。

**场景四：不想踩 Hermes 的那些坑**

记忆紊乱、Skill 打架这些问题在 OpenClaw 里基本不存在。

---

## 5.5 类比：Hermes 是实习生，OpenClaw 是工具

**Hermes 像实习生：**
- 上手快，第三天就能帮你分担工作
- 会自己学习，越用越顺手
- 但有时候会忘事（记忆紊乱）
- 需要你定期指导（清理 Skill）

**OpenClaw 像工具：**
- 刚拿到手要用螺丝刀组装一下
- 组装好了很顺手，每次都按你的方式工作
- 稳定，不会突然失忆
- 但想加新功能得自己动手

---

## 5.6 避坑速查表

### Skills 避坑

| 坑 | Hermes | OpenClaw |
|----|--------|----------|
| Skill 泛滥 | ✅ 定期清理 | ✅ 基本不会 |
| Skill 冲突 | ❌ 只检名字 | ⚠️ 无检语义 |
| Skill 不生效 | ⚠️ snapshot 不同步 | ✅ 下次回复前刷新 |
| Skill 选错 | ⚠️ 随机性 | ⚠️ description 歧义 |

### SubAgent 避坑

| 坑 | Hermes | OpenClaw |
|----|--------|----------|
| 子任务中断 | ⚠️ 无隔离机制 | ✅ 独立 session |
| 结果收集 | ⚠️ 返回值可能不完整 | ⚠️ 要调用 sessions_yield |
| 超时不停止 | ⚠️ 外部 CLI 需手动 kill | ⚠️ 无强制 kill |
| 外部 CLI 桥接 | ✅ claude-code skill | ❌ acpx 格式不兼容 |

### Memory 避坑

| 坑 | Hermes | OpenClaw |
|----|--------|----------|
| 记忆看不到新内容 | ✅ frozen snapshot | ⚠️ CLI 冷启动慢 |
| 多 session 污染 | ⚠️ Provider 并行 | ⚠️ scope 参数可能失效 |
| 并发写入失败 | ⚠️ External Provider | ⚠️ memory.db lock |
| 幽灵 session | ⚠️ Background Review 叠加 | ⚠️ dreaming session |

---

## 5.7 Hermes 必做避坑清单

```
✅ 每两周手动清理一次 Skills（删除长期未用的）
✅ 关闭不用的 External Memory Provider
✅ 不要在一个 session 做太多轮（控制在 60 轮以内）
✅ 做复杂任务时，开新 session
✅ 把高频事实写死在 SOUL.md 里
✅ 定期备份 ~/.hermes/memories/MEMORY.md
❌ 不要让 Skills 数量超过 50 个
❌ 不要同时开两个 Hermes 实例
❌ 不要在 cron job 里写入 MEMORY.md
```

---

## 5.8 OpenClaw 必做避坑清单

```
✅ gateway 启动后做一次 memory warmup
✅ 把高频事实写进 AGENTS.md
✅ 高并发场景下关闭 auto-reply
✅ 定期清理 dreaming session
✅ spawn 子任务后一定要调用 sessions_yield
✅ spawn 前确认 ACP 协议启用
❌ 不要在高并发飞书群里开 auto-reply
❌ 不要在单个 session 里做超过 50 轮
❌ 不要忽略 .acp-stream.jsonl 的异常增长
```

---

## 5.9 最后的忠告

没有完美的系统，只有适合的选择。

**用 Hermes：就别想着完全控制它，接受它有自己的想法，定期维护就行。**

**用 OpenClaw：就别想着省心，配置好了才能用，但用好了很稳定。**

---

## 📦 SKILL：第五章实战精华

本章涉及的 SKILL 文件：

- `SKILLS/HERMES/hermes-openclaw-selection-guide.SKILL.md`
- `SKILLS/HERMES/hermes-solo-pitfalls-checklist.SKILL.md`
- `SKILLS/OPENCLAW/openclaw-solo-pitfalls-checklist.SKILL.md`
- `SKILLS/OPENCLAW/memory-system-comparison-quick-ref.SKILL.md`
