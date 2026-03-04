# MemOS 记忆查询方法（192.168.2.200）与原理简介

本文总结在当前环境中如何查询 MemOS 记忆，以及 MemOS 的核心工作原理。

> 默认主机地址：`192.168.2.200`

---

## 1) 快速确认服务状态

先确认 MemOS API 是否可达：

```bash
curl -sS http://192.168.2.200:18000/health
```

若无 `/health` 路由，可直接测业务接口（见下文 `/product/search`）。

---

## 2) 查询 MemOS 记忆的常用方法

### 方法 A：按语义检索（推荐）

使用 `/product/search`：

```bash
curl -X POST 'http://192.168.2.200:18000/product/search' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Token <MEMOS_API_KEY>' \
  -d '{
    "user_id": "openclaw_linux",
    "query": "WSL openclaw gateway 重启 插件告警",
    "source": "openclaw",
    "memory_limit_number": 5,
    "include_preference": true,
    "preference_limit_number": 3,
    "include_tool_memory": false,
    "tool_memory_limit_number": 3
  }'
```

关键参数说明：
- `user_id`：命名空间，必须与写入时一致（例如 `openclaw_linux`）
- `query`：语义检索文本
- `source`：来源标记（常用 `openclaw`）

---

### 方法 B：用唯一关键词精确确认“是否写入成功”

步骤：
1. 在对话里写入一段带唯一关键词的文本（如 UUID/时间戳）
2. 立刻用 `/product/search` 搜该关键词

示例：

```bash
curl -X POST 'http://192.168.2.200:18000/product/search' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Token <MEMOS_API_KEY>' \
  -d '{
    "user_id": "openclaw_linux",
    "query": "trace-20260304-unique-abc123",
    "source": "openclaw",
    "memory_limit_number": 3,
    "include_preference": false,
    "preference_limit_number": 0,
    "include_tool_memory": false,
    "tool_memory_limit_number": 0
  }'
```

---

### 方法 C：从服务端日志反查写入是否命中

查看 MemOS 容器日志：

```bash
docker logs --tail 300 memos-api-docker 2>&1 | grep -Ei 'product/add|product/search|openclaw_linux|HTTP/1.1" 200'
```

常见判断：
- 出现 `POST /product/add ... 200 OK`：说明写入接口成功
- 出现 `POST /product/search ... 200 OK`：说明查询接口成功

---

## 3) MemOS 原理简介（结合 OpenClaw）

### 3.1 数据流

在典型 OpenClaw + MemOS 场景中：

1. **Recall（检索）**：
   - 在对话前/处理中，插件调用 MemOS 的搜索接口
   - 返回与当前问题相关的记忆片段

2. **Add（写入）**：
   - 对话结束后，插件将本轮消息写入 MemOS
   - 由 MemOS 执行结构化与向量化入库

3. **存储后端**：
   - 图结构关系通常走 Neo4j
   - 向量检索通常走 Qdrant
   - 文本检索与元数据过滤由 MemOS 聚合处理

---

### 3.2 为什么会出现“感觉没写进去”

常见原因：

1. **命名空间不一致**：
   - 写入用 `user_id=A`，查询用 `user_id=B`

2. **接口参数不兼容**（历史常见）
   - 例如 `async_mode` 类型不符合 API 要求导致 422

3. **看错通道**：
   - OpenClaw 本地文件记忆（`MEMORY.md`、`memory/*.md`）
   - 与 MemOS 云端/服务端记忆是两条通道

4. **插件加载异常或配置陈旧**：
   - 重启时出现 stale config / plugin not found，会造成阶段性中断

---

## 4) 建议的排查顺序（实用）

1. `curl /product/search` 先确认 API 可达
2. 确认 `user_id` 一致（例如统一 `openclaw_linux`）
3. 查看 `memos-api-docker` 日志是否有 `POST /product/add 200`
4. 用“唯一关键词写入→搜索”做端到端验收
5. 若失败再检查插件配置与 OpenClaw 重启日志

---

## 5) 最小可复用模板

```bash
# 搜索模板
curl -X POST 'http://192.168.2.200:18000/product/search' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Token <MEMOS_API_KEY>' \
  -d '{
    "user_id": "openclaw_linux",
    "query": "<你的查询词>",
    "source": "openclaw",
    "memory_limit_number": 5,
    "include_preference": true,
    "preference_limit_number": 3,
    "include_tool_memory": false,
    "tool_memory_limit_number": 3
  }'
```

---

如需，我可以再补一份「只看 JSON 字段」的返回结果解读（哪些字段最关键、如何判断 recall 命中质量）。
