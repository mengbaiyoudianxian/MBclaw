# MBclaw Agent 系统评审 — R0

**版本**: r0
**日期**: 2026-06-21
**取代**: `design/agent/AGENT-v2.md`（v2 仍假设 R2 会启用 Agent；r0 更严格）

---

## 0. 一句话

> **MBclaw 是被 Agent 调用的记忆服务，不是 Agent 框架。**
> R0/R1 不存在 Agent。R2 中后期最多 1 个，单步循环。

---

## 1. 是否需要 Agent

### R0：不需要
### R1：不需要
### R2 早期：不需要
### R2 中后期：**可能**需要，最多 1 个

| 假设理由 | 真实判断 |
|---|---|
| AI 要自动记忆 | ❌ HTTP 调用方触发 `/close` 即可 |
| AI 要自动检索 | ❌ `/sessions` 自动注入已覆盖 |
| AI 要代用户做事 | ❌ 是 OpenHands/Claude Code 的职责 |
| AI 要后台整理经验 | ❌ 同步管线 + 软淘汰已足够 |
| AI 要互相协作 | ❌ 协作=外部 Agent 调同一个 MBclaw，非内部多 Agent |

---

## 2. Core Agent 设计

### **不存在。**

所有"自动化"动作均为调用方触发的同步函数：

| 动作 | 触发 | Agent? |
|---|---|---|
| 写 summary/keywords/experiences | POST `/close` | ❌ 函数 |
| 注入 system message | POST `/sessions` | ❌ 函数 |
| 经验软淘汰 | pipeline.close() 顺手 | ❌ SQL |
| FTS5 同步 | SQLite 触发器 | ❌ DB |

---

## 3. Design 备用方案（信号触发，最多 1 个）

### 触发条件（必须**同时**满足）
1. 用户**自己提出**具体的"代我做"场景
2. 现有 Agent 框架确认不能满足
3. 场景能写成 ≤3 行任务描述

任一不满足 → 不启用。

### 设计：**1 个 Agent，单步，无嵌套**

```
ReflectionAgent（唯一候选）
  目的：用户主动请求时，对一段对话或一组 experiences 再加工
  输入：session_id 或 experience_ids
  循环：max_steps = 1
  工具：3 个
    - memory.search(q)               只读
    - memory.read_experiences()      只读
    - memory.write_experience(...)   写（单开关 AUTO_APPROVE_WRITE）
  返回：1 段再加工结果 + 写入 experience id
```

### 永久放弃的 Agent

| 假想 Agent | 否决理由 |
|---|---|
| ExtractorAgent | pipeline.close() 同步做即可 |
| ClassifierAgent | < 500 sessions 不分类 |
| SchedulerAgent | 违反"无调度器"限制 |
| ReviewerAgent (Dual-Key) | 已永久否决 |
| Planner + Executor | 双层网络违反限制 |
| CoordinatorAgent | 单 Agent 不需要 |

**3 个 Agent 是上限，不是目标。实际 1 个即可。**

---

## 4. 通信机制

### 仅 2 条通道
| 通道 | 用途 |
|---|---|
| HTTP REST | 外部 Agent → MBclaw API |
| DB 共享读 | 内部 Agent → SQLite via MemoryRepo |

### 禁止
- 消息队列（Redis/RabbitMQ/Kafka）
- pub/sub / 事件总线
- Agent 互调（循环博弈）
- 分布式 RPC（gRPC/Thrift）
- 共享内存 / 文件 IPC

### 远期"多 Agent 协作"形态

```
Agent A (外部进程) ──┐
                    ├──▶ MBclaw HTTP API ──▶ SQLite
Agent B (外部进程) ──┘
```

永远不在 MBclaw 内部跑多 Agent。

---

## 5. 是否过度设计

**任何在 R0/R1 引入 Agent 的方案 = 过度设计。**

### 判定标准（任一为真即过度设计）
- Agent 数 ≥ 2
- 多步循环（max_steps > 1）
- Agent 间互调
- Agent 持自己的存储
- Agent 自主写记忆无审批
- "为未来扩展性"而做

### 已存在的过度设计（已归档）
| 模块 | 归档 |
|---|---|
| agent_runtime.py (438 行) | Memory drafts/legacy/agent/ |
| sub_agent_coordinator.py | Memory rejected/ |
| dual_key.py | Memory rejected/ |
| auto_mode.py | Memory rejected/ |
| approval_gate.py | Memory drafts/legacy/approval/ |
| task_queue + message_priority | Memory drafts/legacy/queue/ |
| skill_extractor + curator | Memory drafts/legacy/agent/ |

---

## Core / Design / Memory 分流

### Core（R0 实现）
**无任何 Agent。** API、pipeline、memory 全是同步函数。

### Design（信号触发才启用）
- ReflectionAgent（单 Agent，单步，3 工具）
- 触发：用户主动 + 现有框架不能满足 + 场景清晰

### Memory（永久放弃）
- 多 Agent 网络
- 循环博弈（Dual-Key、Reviewer 互评）
- 自主决策（Auto Mode）
- Planner + Executor 双层
- Agent 自带存储
- 内部 pub/sub
- 分布式 Agent
