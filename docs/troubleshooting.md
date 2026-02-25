# OpenClaw + MemOS 故障排查速查表

> 目标：先恢复可用，再定位根因。

## A. 快速体检（1分钟）

```bash
docker ps --format '{{.Names}}\t{{.Status}}\t{{.Ports}}' | grep -E 'memos|qdrant|neo4j'
curl -sS -m 10 -X POST 'http://localhost:18000/product/search' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Token <MEMOS_API_KEY>' \
  -d '{"user_id":"openclaw_linux","query":"health check","source":"openclaw","memory_limit_number":1,"include_preference":true,"preference_limit_number":1,"include_tool_memory":false,"tool_memory_limit_number":1}'
```

判定：
- 容器都在运行
- search 返回 `code:200`

---

## B. 常见故障与修复

### 1) 422 Unprocessable Entity

**现象**
- 插件写入失败
- 日志提示参数校验错误

**高频根因**
- `async_mode` 传成布尔值 `true/false`，而不是字符串

**修复**
- 改为：`"async"` 或 `"sync"`
- 重启相关服务后重试

---

### 2) `Missing 'embedding' in metadata`

**现象**
- MemOS 或向量检索异常

**高频根因**
- 向量维度不一致（例如 1024 vs 1536）

**修复**
1. 删除错误维度的 Qdrant collection
2. 重启 MemOS 触发重建
3. 重新执行历史同步脚本

---

### 3) 能写入但检索不到

**检查顺序**
1. `MEMOS_USER_ID` 与脚本 `user_id` 是否一致
2. `MEMOS_BASE_URL` 是否指向正确实例
3. 是否实际执行过 `node scripts/sync_memories.js`
4. 插件 hook 是否生效（before_agent_start / agent_end）

---

### 4) `gh` 推送失败 / 未登录

**现象**
- `gh repo create` 提示未登录

**修复**
```bash
gh auth login
gh auth status
```

---

## C. 推荐兜底策略

- 检索：**MemOS优先，本地记忆兜底**
- 写入：插件实时写入 + 定时脚本补偿
- 身份：全链路统一 `user_id`（例如 `openclaw_linux`）

---

## D. 变更后必做验证

- [ ] `/product/search` 返回 200
- [ ] 搜索“MemOS 安装 openclaw 插件”能命中历史记忆
- [ ] 同步脚本可执行且无报错
- [ ] OpenClaw 对话一次后，MemOS 有新增记录
