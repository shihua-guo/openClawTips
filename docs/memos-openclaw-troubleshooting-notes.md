# MemOS × OpenClaw 排查记录与经验总结

这篇文档记录一次真实排查：OpenClaw 接入本地 MemOS 后，记忆写入虽然成功，但 `mode=fine` 路径不稳定，最终决定先回退到 `fast` / 基础记忆模式，以保证系统稳定可用。

## 1. 排查背景

当前环境：

- OpenClaw 通过 `memos-cloud-openclaw-plugin` 接入本地 MemOS
- MemOS API 地址：`http://localhost:18000`
- OpenClaw 插件配置中启用了：
  - `addEnabled: true`
  - `recallEnabled: true`

目标原本是让 MemOS 不仅完成基础记忆写入，还能稳定走 `mode=fine`，以支持更细的 skill memory / fine 级别处理。

---

## 2. 初始现象

排查时观察到的典型现象：

- `/product/add` 返回 `200 OK`
- `/product/search` 返回 `200 OK`
- 说明基础写入和检索链路都没有完全断

但同时日志里持续出现：

- `process_skill_memory_fine`
- `History is None in Skills`
- `Not enough messages to extract skill memory`
- `LLM generate failed`
- `No task chunks found`
- `No add/update items prepared`

这说明：

- 请求成功提交给了 MemOS
- fine 路径也确实被触发了
- 但 fine 后续处理没有产出有效记忆项

---

## 3. 先排除的方向

### 3.1 不是 MemOS 服务挂了
验证到：

- MemOS 容器正常运行
- `/product/add` 和 `/product/search` 都可正常返回 200

所以不是“服务不可用导致降级”。

### 3.2 不是 OpenClaw 完全没写入
因为基础写入链路是成功的。

问题不在“有没有写入”，而在“fine 写入有没有稳定产出结构化结果”。

---

## 4. 中间关键发现

### 4.1 插件最初没有显式传 `mode=fine`
检查 OpenClaw 插件代码后发现：

- 插件原本只传了 `async_mode`
- 没有显式传 `mode`
- 默认 `asyncMode: true`

这意味着最初的确有可能默认走了快速路径。

因此做了第一轮修复：

- 给插件增加 `memoryMode`
- 显式传 `payload.mode`
- 配置改成：
  - `asyncMode: false`
  - `memoryMode: "fine"`

### 4.2 即使切到 `fine`，问题仍然存在
修完后再次观察 MemOS 日志，仍然持续出现：

- `History is None in Skills`
- `Not enough messages to extract skill memory`
- `LLM generate failed`
- `No task chunks found`

这说明：

- `mode=fine` 已经真正被触发
- 但问题并没有随着显式传 `mode=fine` 而解决

---

## 5. 进一步实验

### 5.1 将 `captureStrategy` 从 `last_turn` 改为 `full_session`
目的是验证是否只是上下文太短。

修改后仍然观察到：

- `History is None in Skills`
- `Not enough messages to extract skill memory`

说明问题不只是“last_turn 太短”。

### 5.2 手工构造最小 `/product/add` fine 请求
为了排除 OpenClaw 插件 payload 污染问题，直接手工发送最小请求：

```json
{
  "messages": [
    {"role": "user", "content": "我喜欢草莓，也在排查 MemOS fine 模式写入问题。"},
    {"role": "assistant", "content": "已记录：用户喜欢草莓；当前正在排查 MemOS fine 模式与 OpenClaw 插件兼容性。"}
  ],
  "user_id": "openclaw_linux",
  "conversation_id": "debug-minimal-fine",
  "async_mode": "sync",
  "mode": "fine"
}
```

结果：

- 仍然出现同样的 fine 链路错误
- 仍然没有产出有效 add/update items

这一步非常关键，因为它说明：

> 问题主因不在 OpenClaw 插件 payload，而在 MemOS 当前 `mode=fine` 路径本身。

---

## 6. 新增错误信号

除了 skill memory 相关错误，还观察到：

- `Error batch computing embeddings: 'NoneType' object is not iterable`

这说明：

- MemOS 当前 fine 路径里不只是 skill memory 提取不稳
- 连 embedding / multi-modal 相关分支也在吃到异常数据或未兜底情况

---

## 7. 最终结论

本次排查的结论是：

### 结论 1
OpenClaw 插件最初未显式传 `mode=fine`，这确实是一个需要修正的点。

### 结论 2
但即使修正为：

- `async_mode = sync`
- `mode = fine`
- `captureStrategy = full_session`

MemOS 当前 fine 路径仍然无法稳定从普通 messages 中提取 skill memory。

### 结论 3
即使完全绕过 OpenClaw 插件，直接手工发送最小 fine 请求，问题依然存在。

因此可以判断：

> **主问题在 MemOS 当前 `mode=fine` 链路本身，而不是 OpenClaw 插件。**

---

## 8. 当前务实策略

由于目标是先保证记忆系统稳定可用，因此最终采用的策略是：

- 回退到 `fast` / 基础记忆模式
- 不再强求当前环境下的 fine/skill-memory 链路

最终配置为：

```json
{
  "captureStrategy": "last_turn",
  "asyncMode": true,
  "memoryMode": "fast"
}
```

这样做的意义是：

- 保住 `/product/add` 稳定写入
- 保住 `/product/search` 稳定召回
- 避免 fine 链路持续报错与无效消耗

---

## 9. 对后续排查的建议

如果将来还要继续挑战 `mode=fine`，建议不要再优先排 OpenClaw 插件，而是直接进入 **MemOS 内核调试**：

重点查看：

- `process_skill_memory_fine`
- `_preprocess_extract_messages`
- `_split_task_chunk_by_llm`
- `multi_modal_struct`

同时检查：

- `MEMRADER_MODEL`
- `MOS_CHAT_MODEL`
- 当前模型输出格式是否符合 MemOS fine 链路预期

---

## 10. 一句话经验

这次排查最重要的经验可以总结成：

> **当 OpenClaw × MemOS 的基础写入正常、但 fine 路径持续报 `History is None / Not enough messages / No task chunks found` 时，应优先怀疑 MemOS 自身 fine 链路，而不是一味继续折腾 OpenClaw 插件。**
