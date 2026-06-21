# MBclaw 生死级重构评审

**评审人**: Claude（CTO 角色）
**日期**: 2026-06-21
**取代**: 无（首版生死评审）

---

## 1. 项目结论

### **值得做。但当前形态必须死。**

| 维度 | 判断 |
|---|---|
| 问题真实性 | ✅ 真问题——"AI 无长期记忆"是所有 LLM 应用的核心痛点 |
| 差异化 | ⚠️ 弱——mem0 / Letta / Zep / Cognee 已在场；MBclaw 无清晰护城河 |
| 当前实现 | ❌ 必须推翻——10379 行里 7000+ 行是装饰品 |
| 团队产能 | ❌ 被自己拖垮——39 services 单人维护不可能 |
| 商业可能 | 🟡 未知——无用户、无付费意愿验证 |

**裁决**：保留方向，杀死现状。当前 Lite 视为"昂贵的 prototype"，抽 5 个 service 重写，其余归档。

---

## 2. MVP 核心（≤5）

只做一件事：**让 AI 跨会话不失忆**。

| # | 能力 | 边界 |
|---|---|---|
| C1 | 对话持久化 | 只存 role+content+ts |
| C2 | 会话摘要 | 同步 LLM，≤300 字 |
| C3 | 关键词索引 | jieba + TF-IDF + FTS5 |
| C4 | 跨会话检索 | FTS5 + 关键词，写死打分 |
| C5 | 新会话注入 | top-3 摘要 + top-2 失败教训，≤800 字 |

预算：≤1500 行 / ≤5 业务表 / ≤5 端点 / 1 周。

---

## 3. 删除项（→ Memory）

| 模块 | 体积 | 删除理由 |
|---|---|---|
| Utopia（乌托邦计划） | ~770 行 | 偏离使命 |
| Psychology Engine | 333 行 | 伪科学评分 |
| Thought Collision | 272 行 | 创意非记忆 |
| Dual-Key 协作 | 96 行 | 同模型互评偏见 |
| Sub-Agent Coordinator | 131 行 | 给虚构 Agent 设计 |
| Auto Mode | 103 行 | 安全边界未定义 |
| MiMo 特殊化 | 194 行 | 单 provider 锁死 |
| SkillExtractor + Curator | ~470 行 | 无真实 Agent |

共 ~2400 行进 Memory 归档，不进 R1 构建。

---

## 4. 延期项（→ Design）

| 模块 | 触发信号 |
|---|---|
| MEMORY.md 双态架构 | C5 命中率 < 60% |
| Project DNA | 多项目混淆 |
| 向量检索 | C4 召回率 < 70% |
| Agent Runtime（重写） | 用户具体场景请求 |
| 工具索引三层 | Agent + 工具 > 10 |
| 多模型调度 | ≥3 provider |
| 写入审批门（多维） | Agent 写入出错过 |
| 空闲调度器 | Session > 100 |
| 树状分类 | Session > 100 + 关键词失效 |
| Reflection / Dreaming | 摘要质量 < 75% |
| 11 平台网关 | 单平台 DAU > 50 |
| i18n 完整 | 海外用户 ≥10 |
| miclaw / Android | Lite 稳定 3 个月 |
| K8s | 单机扛不住 |
| MBclaw-Full 完整愿景 | R2 完成 + PMF |

15 项延期，信号触发，无固定排期。

---

## 5. 架构风险 Top 3

### 🔴 风险 #1：先造完了再找用户

39 services / 24 models 全部零用户阶段建成。文档声称"33/34 完成"无端到端验证。每加一 service 重构成本 ×1.05，已进指数后段。

**修复**：R0 冻结、R1 重起干净分支、删除"无验证完成"声明、强制 "demo 录屏 + 调用日志" 准入。

### 🔴 风险 #2：记忆能力碎片化，无统一抽象

`memory_store / memory_service / summary_service / action_memory_service / skill_extractor / classification_service / vector_store / layered_search / dna_service` 9 处并行，互不一致。

**修复**：R1 强制 `MemoryRepo` 抽象层，业务零直连存储/LLM。

### 🔴 风险 #3：无 Agent 但全身都是 Agent 假设

approval_gate / auto_mode / sub_agent_coordinator / dual_key / task_queue / message_priority 都假设有 Agent 自主写记忆——实际没有。30%+ 代码服务于不存在的需求。

**修复**：R1 删除所有"Agent 写记忆"防御，简化为单一开关 `AUTO_APPROVE_WRITE`。

---

## 6. 是否方向错误

### **方向正确，形态错误，差异化薄弱。**

- ✅ 方向：长期记忆是真问题
- ❌ 形态：当前像"AI 操作系统"，应做"记忆服务中间件"
- ⚠️ 差异化：必须 R2 前回答"为什么不用 mem0"

### 定位收窄（CTO 强决策）

> **MBclaw 不是"长期记忆智能体平台"，是"中文优先的 LLM 长期记忆服务"。**
> Agent 由别人写（OpenHands / Claude Code），MBclaw 提供记忆。

R1 演示目标变更：从"AI 记得用户" → "任何 Agent 框架接入 MBclaw 后立刻获得跨会话记忆"。

---

## 总裁决

| 维度 | 结论 |
|---|---|
| 值得做 | ✅ |
| 方向错误 | ❌（方向对，**定位需收窄**） |
| 推翻当前实现 | ✅ |
| MVP 重定义 | ≤5 能力，≤1500 行，1 周 |
| 立即停止 | 39 services / 13 项目 / 自我汇报式完成度 |
| 必须开始 | 真实用户调用 / 端到端验证 / 竞品差异化 |
