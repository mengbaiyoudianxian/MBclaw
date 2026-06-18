# 08 — OpenClaw 参考分析

> OpenClaw 是一个拥有 379K+ star 的个人 AI 助手系统，由 Peter Steinberger 创建。
> 仓库: `openclaw/openclaw` — TypeScript, MIT License
> 其记忆系统是 MBclaw 最重要的参考对象。

## OpenClaw 是什么

本地优先、多渠道 AI 网关。以守护进程形式运行在用户自己的设备上，
通过 WhatsApp/Telegram/Slack/Discord/Signal/iMessage 等 16+ 渠道与用户交互。

| 维度 | OpenClaw | MBclaw-Lite (当前) |
|------|----------|---------------------|
| 语言 | TypeScript | Python |
| 架构 | Gateway Daemon + WebSocket + Plugin | FastAPI REST + SQLite |
| 记忆模型 | **三层记忆**: MEMORY.md / daily / dreams | 单层 SQLite 存储 |
| 会话模型 | 持久化 JSONL transcript + 压缩 + 清理 | Session → Messages 表 |
| 多智能体 | 一等公民，完全隔离 | 单用户预留多用户 |
| 插件系统 | 丰富 SDK + 类型化 Hooks + ClawHub 市场 | 无 |
| 技能系统 | SKILL.md + YAML frontmatter + 门控 | 无 |

## MBclaw 应该借鉴的核心模式

### 1. 三层记忆模型（最关键）

```
OpenClaw 记忆体系:
  MEMORY.md          ← 持久、精选、紧凑。每次 DM 会话启动时加载。
                       存事实、偏好、长期决策。
  
  memory/YYYY-MM-DD.md ← 每日工作笔记。今天+昨天自动加载。
                          语义搜索 via memory_search / memory_get。
  
  DREAMS.md          ← 后台整合日记。自动从短期提升到长期。
```

**对比 MBclaw 当前**：所有数据存 SQLite，没有分层。
**改进方向**：增加文件级记忆缓存层（MEMORY.md + daily notes）。

### 2. Memory Flush（记忆冲刷）

在上下文压缩前，一次静默轮次提示 Agent 将重要上下文保存到磁盘。
可使用更便宜的模型执行。

**工作流**：
1. 会话接近上下文限制
2. 触发 Memory Flush → 静默调用 LLM "保存重要信息"
3. 写入 MEMORY.md / daily notes
4. 触发 Compaction → 压缩旧轮次
5. 再次 Memory Flush → 保存压缩后的要点

**对比 MBclaw 当前**：Session complete 时一次性生成总结。
**改进方向**：分为 Flush（保存）→ Compact（压缩）→ Flush（再保存）三个阶段。

### 3. Dreaming（梦想整合）

后台整合通道，选择性地将短期记忆提升为长期记忆。

**工作流**：
1. 收集短期信号（最近的 daily notes、总结、关键词）
2. 评分候选记忆（分数、召回频率、查询多样性）
3. 通过质量阈值筛选
4. 将合格的候选提升到 MEMORY.md

**对比 MBclaw 当前**：无后台整合机制。
**改进方向**：增加 Dreaming 服务，定期从 sessions/summaries 中提取并整合到 Project DNA。

### 4. 持久化 JSONL Session Transcript

每个会话一个 `<sessionId>.jsonl` 文件，记录完整对话历史。
- 可审计、可搜索、可重放
- 文件级写锁防止竞态
- 支持 truncateAfterCompaction 模式

**对比 MBclaw 当前**：Messages 存在 SQLite 中。
**改进方向**：同时输出 JSONL 文件作为不可变审计日志。

### 5. 行动感知记忆

不只要记住"说了什么"，还要记住行动的约束条件：
- 权限边界
- 时间敏感度
- 过期时间
- 来源权威性

### 6. 技能系统

SKILL.md 格式 + YAML frontmatter：
- `requires.bins` / `requires.env` / `requires.config` — 自动门控
- `user-invocable` / `disable-model-invocation` — 调用控制
- `command-dispatch` / `command-tool` — 命令路由
- 加载顺序: workspace → project → personal → managed → bundled

## 不应该照搬的

| OpenClaw 特性 | 不适合 MBclaw 的原因 |
|--------------|---------------------|
| TypeScript 技术栈 | MBclaw 定位 Python 生态 (OpenHands/LangGraph) |
| 多渠道 Gateway | MBclaw 是记忆系统不是消息网关 |
| 设备配对/身份 | MBclaw 是单用户 MVP |
| Docker/SSH 沙盒 | MBclaw 用 proot (Android 约束) |
| WebSocket 协议 | REST 更简单，适合 MVP |

## 优先级排序

| 优先级 | 特性 | 理由 |
|--------|------|------|
| **P0** | 三层记忆 (MEMORY.md + daily + dreams) | 直接回答"长期记忆"问题 |
| **P0** | Memory Flush (压缩前保存) | 防止上下文丢失 |
| **P0** | Dreaming (后台整合) | 短期→长期自动提升 |
| **P1** | JSONL Session Transcript | 可审计的不可变日志 |
| **P1** | 行动感知记忆 | 捕获权限/时间/来源约束 |
| **P2** | 技能系统 | 扩展性，未来阶段 |
| **P3** | 多智能体路由 | 当前不需要 |
