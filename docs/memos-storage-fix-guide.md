# MemOS 记忆存储失败排查与修复指南

> **适用环境**：MemOS Docker 部署 + OpenClaw 插件集成
> **适用版本**：MemOS (memos-dev-memos image)，Docker Compose v2
> **典型症状**：`/product/add` 返回 200 但 Qdrant 中 0 条数据，`/product/search` 返回空 `text_mem`

---

## 问题概述

OpenClaw 的记忆没有成功存储到 Qdrant 中。表现为：

- `/product/add` API 返回 `{"code":200,"message":"Memory added successfully","data":[]}` —— 注意 `data` 为空数组
- `/product/search` 返回的 `text_mem` 始终为空
- Qdrant 集合 `neo4j_vec_db` 的 `points_count` 为 0

最终发现共有 **5 个问题** 需要修复。

---

## 排查与修复步骤

### 第 1 步：检查容器运行状态

```bash
docker ps --format '{{.Names}}\t{{.Status}}\t{{.Ports}}' | grep -E 'memos|qdrant|neo4j|rabbitmq'
```

确认以下容器都在运行：
- `memos-api-docker` → 端口 8000
- `qdrant-docker` → 端口 6333
- `neo4j-docker` → 端口 7474/7687

**发现问题**：缺少 `rabbitmq-docker` 容器。

---

### 第 2 步：检查 MemOS 组件初始化状态

```bash
docker logs --tail 20 memos-api-docker 2>&1 | grep -E "initialized|WARNING|startup"
```

**关键日志**：
```
WARNING - mem_scheduler.py:253 - validate_partial_initialization - 
Failed to initialize components: openai, graph_db. Successfully initialized: rabbitmq
```

这说明 MemOS 有 3 个核心组件需要初始化：
1. `rabbitmq` —— 消息队列（异步处理记忆）
2. `openai` —— LLM（用于分析/提取记忆）
3. `graph_db` —— 图数据库（存储记忆关系）

任何一个初始化失败，记忆都**不会**被实际处理和存储。

---

### 第 3 步：修复 Embedding 配置重复（问题 ①）

```bash
cd ~/MemOS
grep -n "MOS_EMBEDDER" .env
```

**问题**：`.env` 文件中有多个 `MOS_EMBEDDER_API_BASE` 条目，旧的 `localhost:8000`（自引用，错误）覆盖了正确值。

**修复**：注释掉旧的 Embedder 配置，保留正确的配置：

```bash
# 在 .env 中找到旧的 MOS_EMBEDDER 条目并注释掉，例如：
# MOS_EMBEDDER_API_BASE=http://localhost:8000/v1  ← 错误，注释掉
# MOS_EMBEDDER_MODEL=text-embedding-v3            ← 如果重复，注释掉

# 确保保留正确的配置（已经有就不用重复添加）：
MOS_EMBEDDER_BACKEND=universal_api
MOS_EMBEDDER_PROVIDER=openai
MOS_EMBEDDER_MODEL=text-embedding-v3
MOS_EMBEDDER_API_BASE=http://192.168.2.200:3000/v1
MOS_EMBEDDER_API_KEY=<你的 API Key>
```

> ⚠️ **注意**：`.env` 文件中同名的环境变量，后面的值会覆盖前面的。如果有重复条目，务必注释掉错误的那一行。

---

### 第 4 步：添加 RabbitMQ 容器（问题 ②）

MemOS 依赖 RabbitMQ 做**异步消息处理**。当调用 `/product/add` 时，API 立即返回 200，但实际的记忆分析、embedding 生成和存储都是通过 RabbitMQ 消息队列在后台异步执行的。如果 RabbitMQ 不可用，记忆就不会被实际处理。

#### 4.1 修改 docker-compose.yml

在 `docker/docker-compose.yml` 的 `services:` 下添加 RabbitMQ 服务：

```yaml
  rabbitmq:
    image: rabbitmq:3-management
    container_name: rabbitmq-docker
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    restart: unless-stopped
    networks:
      - memos_network
```

同时在 `memos` 服务的 `depends_on` 中加上 `rabbitmq`：

```yaml
  memos:
    ...
    depends_on:
      - neo4j
      - qdrant
      - rabbitmq    # ← 新增
```

#### 4.2 修改 .env 中的 RabbitMQ 配置

```bash
# RabbitMQ 连接配置（注意 HOST_NAME 必须是 Docker 容器名）
MEMSCHEDULER_RABBITMQ_HOST_NAME=rabbitmq-docker
MEMSCHEDULER_RABBITMQ_USER_NAME=guest
MEMSCHEDULER_RABBITMQ_PASSWORD=guest
MEMSCHEDULER_RABBITMQ_VIRTUAL_HOST=memos
MEMSCHEDULER_RABBITMQ_ERASE_ON_CONNECT=true
MEMSCHEDULER_RABBITMQ_PORT=5672
```

> ⚠️ **注意**：`HOST_NAME` 不能写 `localhost`，因为在 Docker 网络中 `localhost` 指向容器自身，而不是宿主机或其他容器。必须使用 Docker 容器名 `rabbitmq-docker`。

#### 4.3 启动 RabbitMQ 并创建 vhost

```bash
# 启动所有容器
sudo docker compose -f docker/docker-compose.yml up -d

# 创建 memos 虚拟主机
sudo docker exec rabbitmq-docker rabbitmqctl add_vhost memos

# 设置 guest 用户对 memos vhost 的权限
sudo docker exec rabbitmq-docker rabbitmqctl set_permissions -p memos guest ".*" ".*" ".*"
```

---

### 第 5 步：修复 OpenAI LLM 组件（问题 ③）

MemOS 的环境变量使用**特殊前缀**，不是标准的 `OPENAI_API_KEY`。

#### 5.1 找到正确的环境变量前缀

通过阅读源码 `src/memos/configs/mem_scheduler.py`，发现 MemOS 使用 `EnvConfigMixin` 系统来读取配置：

```python
class OpenAIConfig(BaseConfig, DictConversionMixin, EnvConfigMixin):
    api_key: str = Field(default="", description="API key for OpenAI service")
    base_url: str = Field(default="", description="Base URL for API endpoint")
    default_model: str = Field(default="", description="Default model to use")
```

可以通过以下命令确认前缀：

```bash
docker exec memos-api-docker python3 -c \
  "from memos.configs.mem_scheduler import OpenAIConfig; print(OpenAIConfig.get_env_prefix())"
# 输出: MEMSCHEDULER_OPENAI_
```

#### 5.2 在 .env 中添加正确的配置

```bash
# MemOS OpenAI LLM 组件（前缀: MEMSCHEDULER_OPENAI_）
MEMSCHEDULER_OPENAI_API_KEY=<你的 API Key>
MEMSCHEDULER_OPENAI_BASE_URL=http://192.168.2.200:3000/v1
MEMSCHEDULER_OPENAI_DEFAULT_MODEL=qwen3.5-flash
```

> ⚠️ **关键点**：
> - 前缀是 `MEMSCHEDULER_OPENAI_`（不是 `OPENAI_`）
> - 字段名来自 Python 类属性：`api_key` → `API_KEY`，`base_url` → `BASE_URL`，`default_model` → `DEFAULT_MODEL`
> - `.env` 中原有的 `OPENAI_API_KEY=dummy` 对 MemOS 的此组件**无效**

---

### 第 6 步：修复 GraphDB 组件（问题 ④）

同样通过源码找到 GraphDB 的配置前缀：

```bash
docker exec memos-api-docker python3 -c \
  "from memos.configs.mem_scheduler import GraphDBAuthConfig; print(GraphDBAuthConfig.get_env_prefix())"
# 输出: MEMSCHEDULER_GRAPHDBAUTH_
```

GraphDB 配置类：

```python
class GraphDBAuthConfig(BaseConfig, DictConversionMixin, EnvConfigMixin):
    uri: str = Field(default="bolt://localhost:7687")
    user: str = Field(default="neo4j")
    password: str = Field(default="", min_length=8)
```

在 .env 中添加：

```bash
# MemOS GraphDB 组件（前缀: MEMSCHEDULER_GRAPHDBAUTH_）
MEMSCHEDULER_GRAPHDBAUTH_URI=bolt://neo4j-docker:7687
MEMSCHEDULER_GRAPHDBAUTH_USER=neo4j
MEMSCHEDULER_GRAPHDBAUTH_PASSWORD=12345678
```

> ⚠️ **注意**：`URI` 中的主机名必须是 Docker 容器名 `neo4j-docker`（不是 `localhost`），密码必须与 `docker-compose.yml` 中 `NEO4J_AUTH` 设置一致。

---

### 第 7 步：重启 Docker 容器

修改完所有 `.env` 配置后，需要**强制重建**容器使环境变量生效：

```bash
cd ~/MemOS

# 强制重建所有容器（推荐）
sudo docker compose -f docker/docker-compose.yml up -d --force-recreate

# 或者只重启 memos-api（如果只改了 .env 中跟 memos 相关的配置）
sudo docker compose -f docker/docker-compose.yml up -d --force-recreate memos
```

> ⚠️ **为什么必须用 `--force-recreate`？**
> 普通 `restart` 不会重新读取 `env_file` 中的变量（只会使用容器创建时的环境变量）。只有 `--force-recreate` 才会销毁旧容器并用新的环境变量创建新容器。

等待 15 秒后验证初始化状态：

```bash
sleep 15
docker logs --tail 10 memos-api-docker 2>&1 | grep -E "initialized|WARNING|startup"
```

**期望结果**：不再出现 `Failed to initialize components` 警告，只看到：

```
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

---

### 第 8 步：验证存储是否正常

#### 8.1 正确的 API 请求格式（问题 ⑤）

MemOS 的 `/product/add` 必须使用 `messages` 数组格式，**不能**使用 `memory` 字符串格式：

```bash
# ❌ 错误格式（会被 SimpleStruct 跳过，data 返回空数组）
curl -X POST 'http://127.0.0.1:8000/product/add' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Token test' \
  -d '{
    "user_id": "openclaw_linux",
    "memory": "Alan likes Python",
    "source": "openclaw",
    "cube_types": ["text_mem"]
  }'
# 返回: {"code":200,"message":"Memory added successfully","data":[]}
# 注意 data 是空的！记忆没有被存储！

# ✅ 正确格式（使用 messages 数组）
curl -X POST 'http://127.0.0.1:8000/product/add' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Token test' \
  -d '{
    "user_id": "openclaw_linux",
    "messages": [
      {"role": "user", "content": "I use Python for automation"},
      {"role": "assistant", "content": "Great, Python is excellent for automation tasks."}
    ],
    "info": {"source_type": "openclaw"},
    "cube_types": ["text_mem"]
  }'
# 返回: {"code":200,"message":"Memory added successfully","data":[{"memory":"...","memory_id":"xxx","memory_type":"LongTermMemory","cube_id":"openclaw_linux"}]}
# data 包含了实际存储的记忆！
```

> ⚠️ **判断标准**：如果 `data` 字段返回空数组 `[]`，说明记忆**没有被处理**。正确存储时 `data` 会包含 `memory_id` 等信息。

#### 8.2 验证 Qdrant 数据

```bash
# 检查 Qdrant 中的记录数
curl -s http://127.0.0.1:6333/collections/neo4j_vec_db | python3 -c \
  "import json,sys; d=json.load(sys.stdin); print('Points:', d['result']['points_count'])"

# 查看所有存储的记忆
curl -s -X POST 'http://127.0.0.1:6333/collections/neo4j_vec_db/points/scroll' \
  -H 'Content-Type: application/json' \
  -d '{"limit":100,"with_payload":true,"with_vector":false}' | python3 -m json.tool
```

#### 8.3 验证搜索功能

```bash
curl -s -X POST 'http://127.0.0.1:8000/product/search' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Token test' \
  -d '{
    "user_id": "openclaw_linux",
    "query": "Python automation",
    "source": "openclaw",
    "memory_limit_number": 5,
    "include_preference": true,
    "preference_limit_number": 3,
    "include_tool_memory": false,
    "tool_memory_limit_number": 3
  }' | python3 -m json.tool
```

---

### 第 9 步：设置开机自启

```bash
# Docker 服务开机自启
sudo systemctl enable docker

# 设置容器重启策略（unless-stopped: Docker 启动时自动恢复容器）
sudo docker update --restart unless-stopped memos-api-docker neo4j-docker qdrant-docker rabbitmq-docker
```

---

## 完整 .env 配置参考

以下是需要在 `~/MemOS/.env` 中确保存在的关键配置（修改后需要 `--force-recreate` 容器）：

```bash
# ========== Embedding 配置 ==========
MOS_EMBEDDER_BACKEND=universal_api
MOS_EMBEDDER_PROVIDER=openai
MOS_EMBEDDER_MODEL=text-embedding-v3
MOS_EMBEDDER_API_BASE=http://192.168.2.200:3000/v1
MOS_EMBEDDER_API_KEY=<你的 API Key>

# ========== RabbitMQ 配置 ==========
MEMSCHEDULER_RABBITMQ_HOST_NAME=rabbitmq-docker
MEMSCHEDULER_RABBITMQ_USER_NAME=guest
MEMSCHEDULER_RABBITMQ_PASSWORD=guest
MEMSCHEDULER_RABBITMQ_VIRTUAL_HOST=memos
MEMSCHEDULER_RABBITMQ_ERASE_ON_CONNECT=true
MEMSCHEDULER_RABBITMQ_PORT=5672

# ========== OpenAI LLM 组件 ==========
# 前缀: MEMSCHEDULER_OPENAI_（不是 OPENAI_）
MEMSCHEDULER_OPENAI_API_KEY=<你的 API Key>
MEMSCHEDULER_OPENAI_BASE_URL=http://192.168.2.200:3000/v1
MEMSCHEDULER_OPENAI_DEFAULT_MODEL=qwen3.5-flash

# ========== GraphDB (Neo4j) 组件 ==========
# 前缀: MEMSCHEDULER_GRAPHDBAUTH_
MEMSCHEDULER_GRAPHDBAUTH_URI=bolt://neo4j-docker:7687
MEMSCHEDULER_GRAPHDBAUTH_USER=neo4j
MEMSCHEDULER_GRAPHDBAUTH_PASSWORD=12345678
```

---

## 问题总结

| # | 问题 | 根因 | 修复方式 |
|---|------|------|----------|
| ① | Embedding 配置失效 | `.env` 中有重复的 `MOS_EMBEDDER_API_BASE`，旧值覆盖了正确值 | 注释掉旧条目 |
| ② | RabbitMQ 缺失 | `docker-compose.yml` 中没有 RabbitMQ 容器 | 添加 `rabbitmq:3-management` 容器 + 创建 vhost |
| ③ | OpenAI 组件初始化失败 | 使用了 `OPENAI_API_KEY` 而非 `MEMSCHEDULER_OPENAI_API_KEY` | 用正确前缀 `MEMSCHEDULER_OPENAI_*` 设置 |
| ④ | GraphDB 组件初始化失败 | 缺少 `MEMSCHEDULER_GRAPHDBAUTH_*` 配置 | 添加 Neo4j 连接配置 |
| ⑤ | API 请求格式错误 | 使用 `"memory":"text"` 被 SimpleStruct 跳过 | 改用 `messages` 数组格式 |

---

## 诊断技巧

### 查看组件初始化状态

```bash
docker logs --tail 20 memos-api-docker 2>&1 | grep -E "initialized|WARNING"
```

- ✅ 正常：只看到 `Application startup complete.`
- ❌ 异常：`Failed to initialize components: openai, graph_db`

### 查看环境变量前缀

每个配置类的 env 前缀可通过以下方式获取：

```bash
docker exec memos-api-docker python3 -c \
  "from memos.configs.mem_scheduler import OpenAIConfig, RabbitMQConfig, GraphDBAuthConfig; \
   print('OpenAI:', OpenAIConfig.get_env_prefix()); \
   print('RabbitMQ:', RabbitMQConfig.get_env_prefix()); \
   print('GraphDB:', GraphDBAuthConfig.get_env_prefix())"
```

### 验证容器内的环境变量

```bash
# 检查容器是否正确读取了 .env
docker exec memos-api-docker env | grep MEMSCHEDULER
docker exec memos-api-docker env | grep MOS_EMBEDDER
```

### 检查 Qdrant 存储详情

```bash
# 查看集合信息（points_count、vector dimension 等）
curl -s http://127.0.0.1:6333/collections/neo4j_vec_db | python3 -m json.tool

# 滚动查看所有存储的记忆
curl -s -X POST 'http://127.0.0.1:6333/collections/neo4j_vec_db/points/scroll' \
  -H 'Content-Type: application/json' \
  -d '{"limit":100,"with_payload":true,"with_vector":false}' | python3 -m json.tool
```
