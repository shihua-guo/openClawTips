# MemOS 安装与 OpenClaw 插件集成（详细实战版）

> 本文基于两类信息整理：
> 1) MemOS 内历史记忆检索结果（真实 API 查询）
> 2) 本地 `memory/*.md` 运维日志
>
> 目标：从零到可用，完成 **MemOS + OpenClaw** 的部署、联调、同步与排障。

---

## 1. 总体架构

核心组件：

- **MemOS API**：对外提供记忆检索/写入接口（示例端口 `18000`）
- **Qdrant**：向量数据库（示例端口 `6333`）
- **Neo4j**：图数据库（示例端口 `7474`）
- **OpenClaw + MemOS 插件**：
  - `before_agent_start`：查询 MemOS（recall）
  - `agent_end`：写入 MemOS（add）
- **sync 脚本**（如 `scripts/sync_memories.js`）：把 `MEMORY.md` 和 `memory/*.md` 批量同步到 MemOS（兜底机制）

---

## 2. 前置准备

### 2.1 环境要求

- Linux 主机（Debian/Ubuntu 等）
- Docker / Docker Compose
- OpenClaw 已安装并可运行
- 可访问 MemOS 插件目录（示例：`~/.openclaw/extensions/memos-cloud-openclaw-plugin`）

### 2.2 建议先确认端口

- MemOS: `18000`
- Qdrant: `6333`
- Neo4j: `7474`

检查容器（示例）：

```bash
docker ps --format '{{.Names}}\t{{.Ports}}' | grep -E 'memos|qdrant|neo4j'
```

---

## 3. 部署 MemOS 与依赖组件（Docker，从零开始）

> 你当前环境已验证过：
> - `memos-api-docker` 在 `18000` 对外服务
> - `qdrant` 在 `6333`
> - `neo4j-docker` 在 `7474`
>
> 下面给的是可直接复用的“从零部署”步骤。

### 3.1 创建部署目录

```bash
mkdir -p ~/memos-stack
cd ~/memos-stack
```

### 3.2 准备 `.env`（容器参数）

在 `~/memos-stack/.env` 写入（按需改密码）：

```env
# 端口映射
MEMOS_PORT=18000
QDRANT_PORT=6333
NEO4J_HTTP_PORT=7474
NEO4J_BOLT_PORT=7687

# 凭据（示例）
NEO4J_AUTH=neo4j/12345678
MEMOS_API_KEY=dummy
```

> 提示：
> - 生产环境不要用 `dummy`；请改成真实随机 Token。
> - 如果机器上已有同端口服务，请改端口映射。

### 3.3 创建 `docker-compose.yml`

在 `~/memos-stack/docker-compose.yml` 写入：

```yaml
services:
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    restart: unless-stopped
    ports:
      - "${QDRANT_PORT}:6333"
    volumes:
      - ./data/qdrant:/qdrant/storage

  neo4j:
    image: neo4j:5
    container_name: neo4j-docker
    restart: unless-stopped
    environment:
      - NEO4J_AUTH=${NEO4J_AUTH}
    ports:
      - "${NEO4J_HTTP_PORT}:7474"
      - "${NEO4J_BOLT_PORT}:7687"
    volumes:
      - ./data/neo4j/data:/data
      - ./data/neo4j/logs:/logs

  memos-api:
    # 按你实际使用的镜像名替换（本示例用通用占位）
    image: memos-api:latest
    container_name: memos-api-docker
    restart: unless-stopped
    depends_on:
      - qdrant
      - neo4j
    environment:
      # 以下变量名请以你使用的 MemOS 镜像文档为准
      - MEMOS_API_KEY=${MEMOS_API_KEY}
      - QDRANT_URL=http://qdrant:6333
      - NEO4J_URI=bolt://neo4j:7687
      - NEO4J_USERNAME=neo4j
      - NEO4J_PASSWORD=${NEO4J_AUTH#neo4j/}
    ports:
      - "${MEMOS_PORT}:8000"
```

### 3.4 启动服务

```bash
docker compose up -d
```

### 3.5 检查服务状态

```bash
docker ps --format '{{.Names}}\t{{.Status}}\t{{.Ports}}' | grep -E 'memos|qdrant|neo4j'
```

期望看到（名称可不同）：
- `memos-api-docker`
- `qdrant`
- `neo4j-docker`

### 3.6 首次连通验证（MemOS API）

```bash
curl -X POST 'http://localhost:18000/product/search' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Token <MEMOS_API_KEY>' \
  -d '{
    "user_id": "openclaw_linux",
    "query": "health check",
    "source": "openclaw",
    "memory_limit_number": 3,
    "include_preference": true,
    "preference_limit_number": 3,
    "include_tool_memory": false,
    "tool_memory_limit_number": 3
  }'
```

若返回 `code:200`，表示 API 已可用。

### 3.7 开机自启

由于已配置 `restart: unless-stopped`，宿主机重启后容器会自动拉起。

---

### 3.8 已知兼容性提醒（非常重要）

历史实践中出现过：
- 云版 `MemOS-Cloud-OpenClaw-Plugin` 与本地 Docker 版 MemOS 在 API 路径/参数上不完全一致。

建议：
- 以你本地实例 API 实测行为为准；
- 插件里重点确认 `search` 和 `add` 的接口路径、参数格式，尤其是 `async_mode`。

---

## 4. 安装并配置 OpenClaw 的 MemOS 插件

插件示例路径：

```text
~/.openclaw/extensions/memos-cloud-openclaw-plugin
```

插件关键逻辑：

- `before_agent_start` -> `/product/search`
- `agent_end` -> `/product/add`

### 4.1 配置环境变量（推荐）

在 `~/.openclaw/.env` 中配置：

```env
MEMOS_BASE_URL=http://localhost:18000
MEMOS_API_KEY=<你的 token>
MEMOS_USER_ID=openclaw_linux
MEMOS_RECALL_GLOBAL=true
```

> 注意：
> - `MEMOS_USER_ID` 要和同步脚本一致（见第 5 节）
> - 避免多个 user_id 混用导致“写到 A，查 B”

### 4.2 关键坑：`async_mode` 类型必须正确

实战中出现过 422 错误，原因是 `async_mode` 被发送成布尔值 `true/false`。

正确做法：请求中传字符串：
- `"async"` 或 `"sync"`

在插件实现里，推荐类似处理：

```js
payload.async_mode = cfg.asyncMode ? "async" : "sync";
```

---

## 5. 统一身份：userId 全链路一致

必须统一以下位置：

1. 插件配置（`MEMOS_USER_ID`）
2. 同步脚本（如 `scripts/sync_memories.js`）
3. 手动调试请求里的 `user_id`

建议统一为：

```text
openclaw_linux
```

这样可以保证：
- 检索与写入命中同一命名空间
- 多设备同步时不会出现“看似丢记忆”

---

## 6. 同步历史记忆（脚本兜底）

你当前实践中，脚本同步是关键兜底机制。

手动全量同步：

```bash
node scripts/sync_memories.js
```

同步内容通常包括：
- `MEMORY.md`
- `memory/YYYY-MM-DD.md`

建议再配合定时任务（Cron）周期执行，防止插件实时写入偶发失败时丢增量。

---

## 7. 验证流程（强烈建议每次改配置后执行）

### 7.1 基础连通

- `docker ps` 看容器状态
- `curl /product/search` 看 HTTP 200

### 7.2 检索验证（不写入测试记忆）

直接搜索关键历史主题：

```bash
curl -X POST 'http://localhost:18000/product/search' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Token <MEMOS_API_KEY>' \
  -d '{
    "user_id": "openclaw_linux",
    "query": "MemOS 安装 openclaw 插件",
    "source": "openclaw",
    "memory_limit_number": 5,
    "include_preference": true,
    "preference_limit_number": 3,
    "include_tool_memory": false,
    "tool_memory_limit_number": 3
  }'
```

判断标准：
- 返回 `code=200`
- `text_mem` 有历史内容

### 7.3 OpenClaw 端到端

确认对话触发插件 hook 后，能：
- 在提问前召回记忆（search）
- 在回复后追加记忆（add）

---

## 8. 常见问题与排障

### 8.1 `422 Unprocessable Entity`

高频原因：
- `async_mode` 参数类型错误（布尔而不是字符串）

处理：
- 改成 `"async"` / `"sync"`
- 重启 MemOS 容器与 OpenClaw 进程后重试

### 8.2 `Missing 'embedding' in metadata`

高频原因：
- Qdrant 集合向量维度不匹配（如 1024 vs 1536）

处理：
1. 删除错误维度 collection（示例曾出现 `neo4j_vec_db`）
2. 重启 MemOS 让其按当前模型重建
3. 重新执行同步

### 8.3 检索不到历史记忆

排查顺序：
1. `user_id` 是否一致
2. 查的是不是正确 MemOS 实例（`baseUrl`）
3. 历史数据是否已经执行过 `sync_memories.js`
4. 插件是否实际加载、生效

### 8.4 插件与自建 MemOS API 不兼容

已出现过：
- 云版本插件与本地 Docker 版 MemOS 在 API 路径/参数上存在差异

建议：
- 以本地 API 实际行为为准，定制插件逻辑
- 每次升级后跑一遍“第 7 节验证流程”

---

## 9. 推荐运行策略（当前可落地）

- **检索策略**：MemOS 优先，本地文件兜底
- **写入策略**：插件实时写入 + 脚本定时补偿
- **命名策略**：全链路统一 userId（`openclaw_linux`）
- **运维策略**：
  - 每次改配置后做一次 API 检索验证
  - 每天/每小时运行一次同步脚本（按资源情况）

---

## 10. 一份最小检查清单（Checklist）

- [ ] `memos-api-docker` 运行中，端口 `18000` 可访问
- [ ] `qdrant` 运行中，端口 `6333` 可访问
- [ ] `neo4j-docker` 运行中，端口 `7474` 可访问
- [ ] `MEMOS_BASE_URL` / `MEMOS_API_KEY` / `MEMOS_USER_ID` 已配置
- [ ] `MEMOS_USER_ID` 与同步脚本一致
- [ ] `async_mode` 实际发送为字符串
- [ ] `node scripts/sync_memories.js` 可成功执行
- [ ] 用“MemOS 安装 openclaw 插件”可检索到历史记忆

---

## 附录：说明

本文是“实战复盘文档”，记录的是你当前环境里已经踩过并验证过的路径，不是抽象官方教程。后续如果你改了模型、向量维度或插件版本，建议及时更新本文件。
