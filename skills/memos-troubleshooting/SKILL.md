---
name: memos-troubleshooting
description: Diagnose MemOS integration problems in OpenClaw, including plugin recall/add failures, `/product/add` and `/product/search` verification, MemOS Docker/container health, log-based root cause analysis, fine-vs-fast memory mode issues, payload compatibility checks, and deciding when to fall back to stable fast/basic memory mode. Use only when the task is specifically about troubleshooting MemOS or OpenClaw↔MemOS memory integration problems.
---

# MemOS Troubleshooting

Use this skill only for **排查 MemOS 问题**，例如：

- OpenClaw 接入 MemOS 后无法 recall / add
- `/product/add` 或 `/product/search` 异常
- MemOS Docker / 容器 / 日志异常
- 记忆写入成功但 `mode=fine` 不稳定
- 想判断问题在 OpenClaw 插件还是 MemOS 本身

不要在普通 OpenClaw 配置、GitHub issue、日常文档整理等任务中调用本 skill。

## 核心排查顺序

按下面顺序收敛问题，不要一开始就改很多配置。

### 1. 先确认服务是否活着
优先检查：

- OpenClaw 当前状态
- MemOS 容器是否运行
- MemOS 端口是否正常

典型检查项：

- `openclaw status`
- `docker ps`
- `docker logs --tail 120 memos-api-docker`

目标：先确认是不是基础服务故障。

### 2. 再确认插件配置
重点查看：

- `~/.openclaw/openclaw.json`
- `memos-cloud-openclaw-plugin` 是否启用
- `baseUrl`
- `userId`
- `addEnabled`
- `recallEnabled`
- `captureStrategy`
- `asyncMode`
- `memoryMode`（如果已有）

不要只凭猜测判断当前模式。

### 3. 再看插件实现
如果怀疑 OpenClaw→MemOS 请求构造有问题，优先读：

- `~/.openclaw/extensions/memos-cloud-openclaw-plugin/index.js`
- `~/.openclaw/extensions/memos-cloud-openclaw-plugin/lib/memos-cloud-api.js`

重点看：

- `/product/search` 调用
- `/product/add` 调用
- payload 里到底传了什么字段
- 是否显式传 `mode`
- 是否显式传 `async_mode`

### 4. 再看 MemOS 日志的错误类型
把错误分层，不要混为一谈。

#### A. 基础链路问题
例如：

- 容器没起来
- API 404 / 500
- 鉴权失败
- baseUrl 错误

#### B. recall/add 业务问题
例如：

- `/product/add` 200 但没有产出有效记忆
- `/product/search` 无结果

#### C. fine 链路问题
例如：

- `History is None in Skills`
- `Not enough messages to extract skill memory`
- `LLM generate failed`
- `No task chunks found`
- `No add/update items prepared`

#### D. 结构/兼容问题
例如：

- `info fields can not contain ...`
- embedding / multi-modal 相关 `NoneType` 报错

## 关于 `mode=fine` 的判断原则

如果观察到：

- `/product/add` 200 OK
- `process_skill_memory_fine`
- 但同时又有：
  - `History is None in Skills`
  - `Not enough messages`
  - `LLM generate failed`
  - `No task chunks found`

不要立刻断言是 OpenClaw 插件问题。

应先做两个判断：

1. 插件是否真的显式传了 `mode=fine`
2. 即使绕过插件、手工发送最小 fine 请求，MemOS 是否仍然报同样错误

如果手工最小 fine 请求也失败，则优先判断：

> 问题主因在 MemOS 当前 fine 链路本身，而不是 OpenClaw 插件。

## 推荐对照实验

### 实验 1：插件自然写入
观察真实系统下的 `/product/add` 日志表现。

### 实验 2：手工最小 fine 请求
用最小 payload 调 `POST /product/add`，例如：

```json
{
  "messages": [
    {"role": "user", "content": "测试消息"},
    {"role": "assistant", "content": "测试回复"}
  ],
  "user_id": "openclaw_linux",
  "conversation_id": "debug-minimal-fine",
  "async_mode": "sync",
  "mode": "fine"
}
```

如果这也失败，说明问题大概率已经进入 MemOS 内部处理链路。

## 务实 fallback 策略

如果用户的目标是“先稳定可用”，而不是“死磕 fine”，优先建议回退到稳定基础模式：

```json
{
  "captureStrategy": "last_turn",
  "asyncMode": true,
  "memoryMode": "fast"
}
```

理由：

- 保持基础写入稳定
- 保持搜索召回可用
- 避免 fine 链路持续报错与无效消耗

## 什么时候继续深挖 MemOS 源码
只有当用户明确要求“必须修好 fine / skill memory”时，才继续深入 MemOS 内核。

重点排查方向：

- `process_skill_memory_fine`
- `_preprocess_extract_messages`
- `_split_task_chunk_by_llm`
- `multi_modal_struct`
- `MEMRADER_MODEL`
- `MOS_CHAT_MODEL`
- 当前模型输出是否符合 MemOS 预期

## 经验总结

排查 MemOS 时，优先顺序应是：

1. 服务活性
2. 插件配置
3. 插件实际请求构造
4. MemOS 日志分层判断
5. 最小对照实验
6. 决定是回退稳定模式，还是进入 MemOS 源码调试

不要在没有做最小对照实验之前，就武断认定责任在 OpenClaw 插件。
