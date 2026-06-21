# MBclaw Agent 系统评审 — R0

**版本**: r0.1（自我修正：移除"永久禁止 Agent"的过度表述）
**日期**: 2026-06-21
**取代**: `design/agent/AGENT-v2.md`

---

## 0. 一句话

> R0/R1 不实装任何 Agent。
> 未来可以演化为功能型 Agent 系统——但必须信号驱动，且不踩 6 个反模式。

---

## 1. 是否需要 Agent

| 阶段 | 判断 |
|---|---|
| R0 | 不需要 |
| R1 | 不需要 |
| R2 早期 | 不需要 |
| R2 中后期 | 可能需要，最多 1 个（ReflectionAgent 候选） |
| R3+ | **可演化**为更完整的功能型 Agent 系统（信号驱动） |

### 当前不需要的理由
| 假设理由 | 真实判断 |
|---|---|
| AI 要自动记忆 | ❌ HTTP 调用方触发 `/close` 即可 |
| AI 要自动检索 | ❌ `/sessions` 自动注入已覆盖 |
| AI 要代用户做事 | ❌ R0/R1 阶段是 OpenHands/Claude Code 的职责 |
| AI 要后台整理经验 | ❌ 同步管线 + 软淘汰已足够 |
| AI 要互相协作 | ❌ R0/R1 协作=外部 Agent 调同一个 MBclaw |

注意：以上"否"都是**当前阶段**的判断，不是永久判断。

---

## 2. Core Agent 设计

### R0/R1：**无任何 Agent**

所有"自动化"动作均为调用方触发的同步函数：

| 动作 | 触发 | Agent? |
|---|---|---|
| 写 summary/keywords/experiences | POST `/close` | ❌ 函数 |
| 注入 system message | POST `/sessions` | ❌ 函数 |
| 经验软淘汰 | pipeline.close() 顺手 | ❌ SQL |
| FTS5 同步 | SQLite 触发器 | ❌ DB |

---

## 3. Design 备用方案（R2 候选）

### R2 启用条件（必须**同时**满足）
1. 用户**自己提出**具体的"代我做"场景
2. 现有 Agent 框架确认不能满足
3. 场景能写成 ≤3 行任务描述

任一不满足 → 不启用。

### R2 候选 Agent：**ReflectionAgent（单步）**

```
ReflectionAgent
  目的：对一段对话或一组 experiences 再加工
  输入：session_id 或 experience_ids
  循环：max_steps = 1
  工具：3 个
    - memory.search(q)               只读
    - memory.read_experiences()      只读
    - memory.write_experience(...)   写（AUTO_APPROVE_WRITE 开关）
  返回：1 段再加工结果 + 写入 experience id
```

### R3+ 可能演化的功能型 Agent（非穷举）

| 候选 | 触发信号举例 |
|---|---|
| ExtractorAgent（重写版） | LLM 同步提取质量不足 |
| ClassifierAgent | sessions > 500 且关键词检索失效 |
| RecallAgent | C5 召回精度成为瓶颈 |
| **新形态（现在想不到的）** | **真实需求出现时再设计** |

**关键**：每个新 Agent 都是**独立论证**，不能搭车合并。

---

## 4. 通信机制

### R0/R1：仅 2 条通道
| 通道 | 用途 |
|---|---|
| HTTP REST | 外部 Agent → MBclaw API |
| DB 共享读 | （未来）内部 Agent → SQLite via MemoryRepo |

### R2+ 引入 Agent 后仍遵守
- 所有 Agent 走 MemoryRepo 操作记忆，不直连 DB
- 所有 Agent 走 llm_client 调 LLM，不直连 provider
- 单 Agent 内部循环可以扩展（max_steps > 1），但需独立论证

---

## 5. 真正禁止的 6 个反模式（**永久**，不随阶段解禁）

不是禁止 Agent，是禁止以下**架构模式**：

| # | 反模式 | 禁止理由 |
|---|---|---|
| 1 | **内部 Agent 互调形成循环**（A→B→A） | 状态机爆炸，无法调试 |
| 2 | **同模型 Dual-Key 互评** | 系统性偏见无证据收益 |
| 3 | **在 MBclaw 进程内同时跑多个 Agent** | 违反单进程限制，并发复杂度爆炸 |
| 4 | **提前为"未来扩展"造 Agent 框架** | 已犯过（438 行 agent_runtime）；规则：先有场景，再有 Agent |
| 5 | **无安全边界的 Auto Mode** | 工程伦理 |
| 6 | **Agent 持自己的存储绕过 MemoryRepo** | 一致性灾难 |

这 6 条**永久有效**——它们是"如何做 Agent"的规则，不是"是否能做 Agent"的禁令。

### 多 Agent 协作的允许形态（**任何时候**都允许）

```
Agent A (外部进程) ──┐
                    ├──▶ MBclaw HTTP API ──▶ SQLite
Agent B (外部进程) ──┘
```

R3+ 即使 MBclaw 内部有功能型 Agent，**多 Agent 协作仍然走 HTTP，不在进程内并发**。

---

## 6. 过度设计判定（仍有效）

任一为真 → 立即停止：
- Agent 数 ≥ 2 在同一进程
- 多步循环但没有具体场景论证
- 命中第 5 节的 6 个反模式之一
- "为未来扩展性"而做

---

## Core / Design / Memory 分流（修正版）

### Core（R0/R1）
**无 Agent**。API/pipeline/memory 全是同步函数。

### Design（R2/R3+ 候选）
- ReflectionAgent（R2 候选）
- ExtractorAgent / ClassifierAgent / RecallAgent / 其他（R3+ 候选）
- 全部信号驱动，独立论证

### Memory（永久放弃）
**架构反模式**，不是"Agent"本身：
- 内部 Agent 循环互调
- 同模型互评博弈
- 进程内多 Agent 并发
- 提前造 Agent 框架
- 无边界 Auto Mode
- Agent 自带存储

---

## 自我修正说明

上一版（AGENT-r0 初稿）写了"**永久禁止**多 Agent 网络与协作"——这是修辞过度。
准确表述是"永久禁止 6 个反模式 + 信号触发可启用功能型 Agent"。
修正记录见 `MBclaw-Memory/logs/2026-06-21_cto-self-correction.md`。
