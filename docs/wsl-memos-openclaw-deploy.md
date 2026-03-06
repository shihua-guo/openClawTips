# WSL 部署独立 MemOS 并接入 OpenClaw（实战记录）

本文记录一次真实落地：在 WSL 中独立部署 MemOS（含 Qdrant/Neo4j），并将 WSL 的 OpenClaw 接入该实例。

## 1. 目标
- WSL 内独立运行：`memos-api` + `qdrant` + `neo4j`
- WSL OpenClaw 通过插件接入本地 MemOS
- 与主机实例隔离，使用 `userId=openclaw_wsl`

## 2. 环境与连接
- WSL SSH：`ssh alan@192.168.2.109 -p 2222`
- OpenClaw 配置路径：`~/.openclaw/openclaw.json`
- MemOS 仓库：`~/MemOS`

## 3. 执行步骤
### 3.1 安装 Docker 组件（WSL）
```bash
sudo apt update
sudo apt install -y docker.io docker-compose docker-compose-v2
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

### 3.2 拉取并启动 MemOS 栈
```bash
git clone https://github.com/MemTensor/MemOS.git ~/MemOS
cd ~/MemOS/docker
```

> 该项目 compose 读取 `~/MemOS/.env`，不是 `~/MemOS/docker/.env`。

初始化：
```bash
cp ~/MemOS/docker/.env ~/MemOS/.env
```

### 3.3 关键配置修正（避免启动失败）
在 `~/MemOS/.env` 中确认：

```env
ENABLE_PREFERENCE_MEMORY=false

NEO4J_URI=bolt://neo4j-docker:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=12345678

QDRANT_HOST=qdrant-docker
QDRANT_PORT=6333
QDRANT_URL=http://qdrant-docker:6333
QDRANT_API_KEY=

# LLM/Embedding（按现网 one-api）
OPENAI_API_BASE=http://192.168.2.200:3000/v1
OPENAI_API_KEY=dummy
MOS_CHAT_MODEL=qwen-plus
MOS_EMBEDDER_API_BASE=http://192.168.2.200:3000/v1
MOS_EMBEDDER_API_KEY=dummy
MOS_EMBEDDER_MODEL=text-embedding-v3
MEMRADER_API_BASE=http://192.168.2.200:3000/v1
MEMRADER_API_KEY=dummy
```

启动：
```bash
cd ~/MemOS/docker
docker compose up -d
```

### 3.4 OpenClaw 插件接入（WSL）
在 `~/.openclaw/openclaw.json` 添加：

```json
{
  "plugins": {
    "entries": {
      "memos-cloud-openclaw-plugin": {
        "enabled": true,
        "config": {
          "baseUrl": "http://127.0.0.1:8000",
          "userId": "openclaw_wsl",
          "addEnabled": true,
          "recallEnabled": true,
          "captureStrategy": "last_turn"
        }
      }
    }
  }
}
```

重启：
```bash
openclaw gateway restart
```

## 4. 验收
- `docker ps` 中三个容器均为 Up：
  - `memos-api-docker`
  - `qdrant-docker`
  - `neo4j-docker`
- `curl http://127.0.0.1:8000/docs` 可返回 Swagger HTML。
- OpenClaw 配置已包含 `memos-cloud-openclaw-plugin`，并指向本地 MemOS。

## 5. 常见坑位
1. **Qdrant SSL 错误**：`QDRANT_URL` 不能留占位值，需明确 `http://qdrant-docker:6333`。
2. **Milvus 误触发**：若未部署 Milvus，需 `ENABLE_PREFERENCE_MEMORY=false`。
3. **compose 读取错 env 路径**：以仓库根 `~/MemOS/.env` 为准。
4. **容器内 localhost 误用**：容器互访应使用服务名（`qdrant-docker`/`neo4j-docker`）。

---

如果后续要把这套改成生产可用，建议再补：
- 持久化卷目录规划
- API token 管理
- 监控与自动重启策略
- 备份（Neo4j/Qdrant）策略
