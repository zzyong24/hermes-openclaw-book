---
name: openclaw-memory-concurrency-guide
description: 解决 OpenClaw Memory 并发问题 — memory.db lock、QMD 冷启动延迟、dreaming session 残留
platform: openclaw
triggers:
  - "OpenClaw Memory 并发问题"
  - "memory.db lock timeout"
  - "QMD 查询慢"
  - "dreaming session 清理"
  - "OpenClaw 记忆丢失"
  - "并发写入 memory"
tags:
  - openclaw
  - memory
  - concurrency
  - qmd
---

# 使用方法

## 三大并发场景应对

### 场景一：memory.db lock timeout

高并发飞书消息同时触发 memory 写入，SQLite busy_timeout（默认 5s）可能不够用。

**诊断：**
```bash
cat ~/.openclaw/logs/gateway.log 2>/dev/null \
  | grep -i "lock\|timeout" | tail -20
```

**解决：**
- 降低 auto-reply 触发频率
- 定期清理 sessions，减少并发

### 场景二：QMD 冷启动延迟

每次 queryMemory() 都要 spawn 子进程 + 加载模型，首次查询可能需要 3-8 秒。

**诊断：**
```bash
time openclaw query-memory --scope session --query "test" 2>/dev/null
```

**解决（最有效）：**
```yaml
# ~/.openclaw/config.yaml
memory:
  qmd:
    mcporter:
      enabled: true
      startDaemon: true  # 保持 QMD 常驻，秒级响应
```

**备选方案：gateway 启动后预热**
```bash
openclaw query-memory --scope session --query "warmup" 2>/dev/null
```

### 场景三：Dreaming session 残留

gateway 重启后，dreaming session 的 transcript 文件可能残留。

**诊断：**
```bash
ls ~/.openclaw/workspace/main/memory/sessions/ \
  | grep "dreaming" | wc -l
```

**清理（删除 3 天以上的）：**
```bash
find ~/.openclaw/workspace/main/memory/sessions/ \
  -name "*dreaming*" -mtime +3 \
  -delete
```

**定期 cron：每周日凌晨 2 点执行**
```
0 2 * * 0 find ~/.openclaw/.../sessions/ -name "*dreaming*" -mtime +3 -delete
```

## QMD + LanceDB 配置示例

### 本地 embedding（推荐，隐私优先）

```yaml
# ~/.openclaw/config.yaml
memory:
  provider: qmd
  qmd:
    embedding:
      type: local
      model: ~/.openclaw/models/embedding-model.gguf
    scope: user
    top_k: 5
```

### 远程 embedding（更快，但有隐私考虑）

```yaml
memory:
  provider: qmd
  qmd:
    embedding:
      type: remote
      remote:
        provider: openai
        model: text-embedding-3-small
        api_key: ${OPENAI_API_KEY}
    scope: user
    top_k: 5
```

## 验证 scope 参数是否生效

⚠️ QMD 的 `--scope` 参数在传给 CLI 时如果格式错误，会被静默忽略，变成全局搜索。

```bash
openclaw query-memory --scope session --query "test" 2>&1 \
  | grep -i "scope\|filter" || echo "⚠️ scope 参数可能被忽略"
```