# OpenClaw GitHub Issue 轮询落地指南

这份文档把当前可工作的 GitHub issue 自动扫描方案整理成一套**可复用、可本地落地**的实现说明。

目标是让另一套 OpenClaw 环境只靠这份文档，就能复现相同能力：

- 定时扫描多个 GitHub 仓库的 open issues
- 只处理指定作者的 issue
- 避免重复唤醒
- 在脚本侧尽量完成过滤、去重、粗分类，减少 Agent token 消耗
- 被唤醒后按统一规则处理 issue（`[AI]` 前缀、默认先回复、不乱提 PR、完成后关闭）

---

## 1. 总体设计

这套方案分成两层：

### A. `scripts/check_issues.sh`
负责**前置处理**：

- 扫描哪些仓库
- 只看 open issues
- 只看指定作者
- 过滤不需要处理的标签
- 根据“最近一条非 AI 评论时间”生成去重 marker
- 对 issue 做轻量粗分类
- 只有发现新 issue 或用户有新补充时才唤醒 Agent

### B. `GITHUB_ISSUE_POLLING.md`
负责**Agent 行为规则**：

- 回复格式要求
- 默认先 comment，不直接改仓库
- 什么时候允许提 PR
- 完成后如何收尾

可以把它理解成：

- **脚本负责：值不值得唤醒**
- **规则文件负责：唤醒后该怎么做**

---

## 2. 适用场景

适用于下面这类需求：

- 你希望 OpenClaw 自动扫描 GitHub issue
- 你希望脚本尽量做更多判断，减少模型 token 消耗
- 你希望用户后续在同一个 issue 下补充评论时，Agent 能继续跟进
- 你希望默认行为是“先在 issue 下回复”，而不是动不动就直接开 PR

---

## 3. 核心行为规则

当前落地方案采用以下规则：

1. **只处理指定作者的 open issues**
2. **每分钟扫描一次**（尽量实时）
3. **不再使用时间窗口限制**，任何时间都扫描
4. **每次 AI 在 issue 下回复，正文第一行必须带 `[AI]`**
5. **默认先在 issue 下回复，不直接改代码 / 提 PR**
6. **只有 issue 明确要求修改代码 / 文档 / 仓库文件 / 提交 PR 时，才允许改仓库**
7. **用户在同一 issue 下继续补充时，必须继续跟进**
8. **issue 明确完成后，需要：**
   - 回复最终结论
   - 添加 `completed` 标签
   - 关闭 issue

---

## 4. 为什么 `[AI]` 前缀是必须的

这套轮询方案的去重逻辑依赖 `[AI]` 标记。

脚本不是简单看“issue 有没有最新评论”，而是看：

- **最近一条不是以 `[AI]` 开头的评论时间**
- 如果没有评论，就用 issue `createdAt`

这样设计的意义是：

- AI 自己回复不应触发再次唤醒
- 用户后续补充信息应该触发再次唤醒

因此：

- 如果 AI 回复**没有** `[AI]` 前缀
- 脚本就可能把 AI 的评论误判成“用户新评论”
- 造成去重失真或后续跟进异常

所以 `[AI]` 不是样式问题，而是**去重协议的一部分**。

---

## 5. Issue 粗分类设计

为了进一步减少 Agent 判断成本，脚本会先做轻量粗分类。

当前推荐 4 类：

### 1) `discussion-only`
纯答疑 / 分析 / 说明类，默认：

- 只在 issue 下回复
- 不改代码
- 不提 PR

常见命中词：

- `不要修改代码`
- `只回答问题`
- `只分析`
- `do not modify`
- `no code changes`

### 2) `plan-only`
先给方案，不落地实现，默认：

- 只给方案
- 不改代码
- 不提 PR

常见命中信号：

- 标签：`Idea` / `Plan Mode`
- 文本：`方案` / `计划` / `proposal` / `design`

### 3) `implementation-request`
明确要求落地修改，默认：

- 允许改代码 / 文档 / 配置
- 允许提 PR

常见命中词：

- `修复`
- `实现`
- `修改代码`
- `提交PR`
- `update docs`
- `patch`

### 4) `mixed-or-unclear`
信息不足或混合需求，默认：

- 先在 issue 下追问 / 分析
- 不直接改仓库

---

## 6. 分类优先级

推荐采用下面的优先级：

1. **标签硬规则**
   - `Idea` / `Plan Mode` → `plan-only`
   - `completed` → 直接过滤，不处理

2. **否定修改约束优先**
   - 如出现：
     - `不要修改代码`
     - `不要提 PR`
     - `只回答问题`
   - 即使文本里也出现“修复 / 修改”等词，也优先判成 `discussion-only`

3. **明确实施请求**
   - 有明确“修复 / 实现 / 改代码 / 提交 PR”时，判 `implementation-request`

4. **方案导向**
   - 方案 / 设计 / proposal → `plan-only`

5. **默认保守**
   - 其余 → `mixed-or-unclear`

这个优先级能有效避免：

- 用户明明写了“不要修改代码”
- Agent 却因为看到“修复”二字就直接开 PR

---

## 7. `check_issues.sh` 的推荐职责

脚本建议承担以下职责：

### 必须做
- 扫描仓库列表
- 限定作者
- 限定 open issues
- 标签过滤
- 去重 marker 生成
- 唤醒 Agent

### 建议做
- issue 粗分类
- 把分类摘要写入 wake message
- 把筛选结果写日志

### 不建议做
- 复杂语义推理
- 最终业务决策
- 复杂 multi-step 任务执行

这些仍然应由 Agent 来完成。

---

## 8. 推荐脚本结构

推荐把脚本设计成：

1. **定义仓库列表**
2. **加锁，防止并发重复运行**
3. **逐仓库列 issue 编号**
4. **逐 issue 拉一次完整详情**
5. **做标签过滤 / marker 计算 / 分类**
6. **生成去重键 `ISSUE_KEY`**
7. **与上次扫描状态比较**
8. **只有变化时才唤醒 Agent**
9. **唤醒成功后更新 state 文件**

重点建议：

- 不要把所有 issue 详情一次性混在一个大 TSV 里做复杂解析
- 更稳的做法是：
  - 先列编号
  - 再逐个 issue 拉详情
  - 单个 issue 异常时只跳过自己，不拖垮整轮

---

## 9. 推荐去重策略

去重键建议采用：

```text
repo + issue号 + 最近一条非 [AI] 评论时间
```

如果没有评论，则使用：

```text
createdAt
```

示例：

```text
shihua-guo/clawdbot#58@2026-03-06T09:46:34Z
```

多个 issue 组合后写入 state 文件，作为“本轮扫描快照”。

下一轮扫描时：

- 如果 `ISSUE_KEY` 没变 → 不唤醒
- 如果变了 → 唤醒

这样能做到：

- 新 issue 会唤醒
- 用户在 issue 下补充信息会再次唤醒
- AI 自己回复不会造成连环唤醒

---

## 10. 推荐唤醒消息内容

不要只发：

```text
检测到待处理 Issue: xxx，请开始执行任务。
```

更好的做法是附带最小必要摘要，例如：

```text
HEARTBEAT_CRON_WAKEUP: 检测到待处理 Issue: clawdbot#58, llm-free#1。
分类摘要：clawdbot#58[discussion-only;explicit-no-change-request]; llm-free#1[plan-only;plan-oriented-request]。
请先读取 /path/to/GITHUB_ISSUE_POLLING.md 并严格按其中规则执行。
注意：默认先在 issue 下用 [AI] 前缀回复；若分类为 discussion-only 或 plan-only，则默认不要改仓库；只有 issue 明确要求修改代码/文档/仓库文件/提交 PR 时才允许改仓库；完成后需添加 completed 并关闭 issue。
```

这样能把大量判断前置给脚本，减少 Agent 重复阅读和推理成本。

---

## 11. `GITHUB_ISSUE_POLLING.md` 应写什么

推荐把规则文件写成下面几部分：

### 1. 目的
说明它是 issue polling 行为规范，不是 skill。

### 2. 脚本与规则文件的职责划分
明确：

- 脚本负责“值不值得唤醒”
- 规则文件负责“唤醒后怎么做”

### 3. Agent 行为规则
至少包含：

- 回复要带 `[AI]`
- 默认只在 issue 下回复
- 只有明确要求才允许修改仓库 / 提 PR
- 用户继续补充时要继续跟进
- 完成后 `completed + close`

### 4. 标签规则
例如：

- `Idea`
- `Plan Mode`
- `completed`

### 5. 降低 token 消耗原则
例如：

- 能脚本做的就不要留给 Agent
- 能在唤醒消息里摘要的就不要让 Agent 重扫

---

## 12. 新增仓库时怎么做

如果你想让脚本扫描新的仓库，通常直接修改：

```bash
REPOS=(
  "shihua-guo/clawdbot"
  "shihua-guo/callchain-analysis"
  "shihua-guo/llm-free"
  "shihua-guo/new-repo"
)
```

但最好同时确认 4 件事：

1. 新仓库是否也适用同一套 issue 处理规则
2. issue 作者筛选是否仍然是同一个人
3. 标签语义是否兼容（如 `Idea` / `Plan Mode` / `completed`）
4. 是否仍适合用同一个 Agent 会话来处理

如果这些都兼容，直接加到 `REPOS` 里通常就够了。

---

## 13. Cron 配置建议

如果目标是尽量实时，可以使用：

```cron
* * * * * /root/clawd/scripts/check_issues.sh
```

说明：

- 每分钟扫描一次
- 不再使用时间窗口限制
- 重复唤醒由脚本去重逻辑控制

如果担心并发，脚本内部要配合 `flock` 锁。

---

## 14. 环境要求

至少需要：

- `gh`（GitHub CLI）
- 可用的 `GH_TOKEN`
- OpenClaw CLI 可调用
- cron 可执行脚本

建议：

- 不要在脚本里硬编码敏感 token
- 优先从系统环境变量或安全配置文件中读取 `GH_TOKEN`

---

## 15. 一个可工作的实现要点总结

如果你要在另一台机器上复现这套能力，最小闭环是：

1. 准备 `GITHUB_ISSUE_POLLING.md`
2. 准备 `scripts/check_issues.sh`
3. 在脚本里配置：
   - `REPOS`
   - 作者过滤
   - state/log/lock 文件路径
4. 配置 cron 每分钟执行
5. 确保 wake message 明确要求 Agent 先读规则文件
6. 确保 Agent 在 issue 回复时始终带 `[AI]`

只要这 6 步具备，就能复现当前这套 issue polling 流程。

---

## 16. 当前方案的优势

- 稳定：单个 issue 异常不应拖垮整轮扫描
- 省 token：尽量把筛选、去重、粗分类前置到脚本
- 可控：行为规则集中在一个规则文件里
- 可扩展：新增仓库主要改 `REPOS`
- 可跟进：用户在 issue 下继续补充时可再次唤醒

---

## 17. 后续可继续优化的方向

1. 在日志里记录更细的 skip reason
2. 针对不同 repo 做 repo-specific 规则
3. 在脚本里加入更精确的 issue 类型摘要
4. 对 wake message 做更紧凑的压缩，进一步减少 token 消耗
5. 增加异常告警（如连续 N 次扫描失败时通知）

---

## 18. 结论

这套方案的核心思想非常简单：

> **脚本尽量重，Agent 尽量轻。**

也就是：

- 脚本负责扫描、过滤、去重、粗分类
- Agent 负责理解 issue 内容、回复、执行明确要求、完成收尾

这样既能保证自动化稳定，也能把 token 消耗压到比较合理的水平。
