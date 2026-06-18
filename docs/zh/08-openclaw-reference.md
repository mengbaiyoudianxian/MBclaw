# 08 — OpenClaw 参考分析

> OpenClaw 是一个拥有 379K+ star 的个人 AI 助手系统，由 Peter Steinberger 创建。
> 仓库：`openclaw/openclaw` — TypeScript，MIT License
> 其记忆系统是 MBclaw 最重要的参考对象。
> 文档：docs.openclaw.ai

## OpenClaw 是什么

本地优先、多渠道 AI 网关。以守护进程形式运行在用户自己的设备上，
通过 WhatsApp / Telegram / Slack / Discord / Signal / iMessage 等 16+ 渠道与用户交互。

| 维度 | OpenClaw | MBclaw-Lite（当前） |
|------|----------|---------------------|
| 语言 | TypeScript | Python |
| 架构 | Gateway Daemon + WebSocket + Plugin | FastAPI REST + SQLite |
| 记忆模型 | **三层记忆**：MEMORY.md / daily / dreams | 三层记忆（已参考实现） |
| 会话模型 | 持久化 JSONL transcript + 压缩 + 清理 | Session → Messages 表 + JSONL |
| 多智能体 | 一等公民，完全隔离 | 单用户预留多用户 |
| 插件系统 | 丰富 SDK + 类型化 Hooks + ClawHub 市场 | 无 |
| 技能系统 | SKILL.md + YAML frontmatter + 门控 | 无 |

## MBclaw 已借鉴的核心模式

### 1. 三层记忆模型（最关键）✅ 已实现

```
OpenClaw 记忆体系：
  MEMORY.md          ← 持久、精选、紧凑。每次会话启动时加载。
                       存事实、偏好、长期决策。

  YYYY-MM-DD.md      ← 每日工作笔记。今天 + 昨天自动加载。
                       语义搜索可用。

  DREAMS.md          ← 后台整合日记。自动从短期提升到长期。
```

**MBclaw 实现**：`app/services/memory_service.py`
- `write_memory_md()` / `read_memory_md()` — Tier 1 持久记忆
- `append_daily_note()` / `read_daily_notes()` — Tier 2 每日笔记
- `append_dream_entry()` / `read_dreams()` — Tier 3 整合日记

### 2. Memory Flush（记忆冲刷）✅ 已实现

在 Session 完成时，自动保存上下文到每日笔记：
- 主题、结论、决定、下一步
- 关键词
- 最近对话片段

**MBclaw 实现**：`memory_flush()` 在 `complete_session` 时自动调用

### 3. Dreaming（梦想整合）✅ 已实现

后台整合通道，将短期记忆提升为长期记忆：
1. 收集最近 7 天的总结和关键词
2. 评分候选记忆（关键词权重 >= 2.0）
3. 通过质量阈值筛选
4. 提升到 MEMORY.md

**MBclaw 实现**：`dream()`，可通过 `POST /api/projects/{id}/memory/dream` 触发

### 4. 持久化 JSONL 会话记录 ✅ 已实现

每个会话一个 `<sessionId>.jsonl` 文件：
- 每条消息追加一行
- fcntl 写锁保护
- Session 完成时写完整记录

**MBclaw 实现**：`app/services/transcript_service.py`

### 5. 行动感知记忆（待实现）

不只要记住"说了什么"，还要记住行动的约束条件：
- 权限边界
- 时间敏感度
- 过期时间
- 来源权威性

## 已纳入 PLANNED 但尚未实现的

### 技能系统

SKILL.md 格式 + YAML frontmatter：
- `requires.bins` / `requires.env` / `requires.config` — 自动门控
- `user-invocable` / `disable-model-invocation` — 调用控制
- 加载顺序：workspace → project → personal → managed → bundled

### 多智能体路由

OpenClaw 的 Bindings 系统：
- 确定性路由（channel，accountId，peer → agent）
- 最具体匹配 + AND 语义
- 完全隔离的 workspace、auth、session

## 不应该照搬的

| OpenClaw 特性 | 不适合 MBclaw 的原因 |
|--------------|---------------------|
| TypeScript 技术栈 | MBclaw 定位 Python 生态（OpenHands / LangGraph） |
| 多渠道 Gateway | MBclaw 是记忆系统不是消息网关 |
| 设备配对 / 身份 | MBclaw 是单用户 MVP |
| Docker / SSH 沙盒 | MBclaw 用 proot（Android 约束） |
| WebSocket 协议 | REST 更简单，适合 MVP |

## 优先级总结

| 优先级 | 特性 | 状态 |
|--------|------|------|
| **P0** | 三层记忆（MEMORY.md + daily + dreams） | ✅ 已实现 |
| **P0** | Memory Flush（压缩前保存） | ✅ 已实现 |
| **P0** | Dreaming（后台整合） | ✅ 已实现 |
| **P1** | JSONL 会话记录 | ✅ 已实现 |
| **P1** | 行动感知记忆 | 🔲 计划中 |
| **P2** | 技能系统 | 🔲 计划中 |
| **P3** | 多智能体路由 | 🔲 远期规划 |

## 参考来源

- GitHub：https://github.com/openclaw/openclaw
- 文档：https://docs.openclaw.ai
- VISION.md：https://raw.githubusercontent.com/openclaw/openclaw/main/VISION.md
- AGENTS.md：https://raw.githubusercontent.com/openclaw/openclaw/main/AGENTS.md
