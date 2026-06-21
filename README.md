# MBclaw / Design

> **仓库定位**：MBclaw 设计系统仓库（Design）。
> 存放所有架构设计、规划文档、未验证方案。
> 验证可执行后迁出至 [MBclaw-Lite](https://github.com/mengbaiyoudianxian/MBclaw-Lite)（Core）。
> 被否决方案归档至 [MBclaw-Memory](https://github.com/mengbaiyoudianxian/MBclaw-Memory)。

## 一句话愿景

> 让 AI 像人一样：**记录经验 → 总结经验 → 检索经验 → 复用经验 → 避免重复犯错**。

不是"更大的上下文窗口"，而是"经验沉淀型长期记忆"。

## 三仓库分工

| 仓库 | 角色 | 内容 |
|---|---|---|
| **MBclaw-Lite** | Core / 生产 | 可运行代码、已验证功能、OpenHands 可直接实现的任务 |
| **MBclaw**（本仓库） | Design / 设计 | 架构、规划、未验证方案、MVP 定义、路线图 |
| **MBclaw-Memory** | Memory / 经验库 | 被否决方案、失败经验、灵感草稿、实验日志 |

## 当前状态（2026-06-21）

由 Claude（CTO 角色）完成审计与裁剪。新设计入口：

### 🔥 R0 当前版本（按此执行）
| 文档 | 用途 |
|---|---|
| [`design/roadmap/DEV-PLAN-r0.md`](design/roadmap/DEV-PLAN-r0.md) | **执行 AI 入口**：18 个任务 + 1 周日程 + 风险清单 |
| [`design/audit/SURVIVAL-REVIEW-2026-06-21.md`](design/audit/SURVIVAL-REVIEW-2026-06-21.md) | 生死级评审：值得做、形态错、定位收窄 |
| [`design/mvp/MVP-r0-1week.md`](design/mvp/MVP-r0-1week.md) | 5 功能 / ≤1500 行 / 1 周 |
| [`design/architecture/ARCH-r0.md`](design/architecture/ARCH-r0.md) | 7 文件单进程架构 + 技术选型 |
| [`design/memory/MEMORY-SYSTEM-r0.md`](design/memory/MEMORY-SYSTEM-r0.md) | 长期记忆系统（5 表 + 2 FTS + experiences） |
| [`design/agent/AGENT-r0.md`](design/agent/AGENT-r0.md) | Agent 评审（R0/R1 无 Agent，6 反模式禁止） |

### 历史版本（保留参考）
| 文档 | 状态 |
|---|---|
| [`design/audit/AUDIT-2026-06-21.md`](design/audit/AUDIT-2026-06-21.md) | 首版审计 |
| [`design/mvp/MVP-v2.md`](design/mvp/MVP-v2.md) | 已被 MVP-r0 取代 |
| [`design/architecture/ARCH-v2.md`](design/architecture/ARCH-v2.md) | 已被 ARCH-r0 取代 |
| [`design/database/SCHEMA-v2.md`](design/database/SCHEMA-v2.md) | 已被 MEMORY-SYSTEM-r0 取代 |
| [`design/agent/AGENT-v2.md`](design/agent/AGENT-v2.md) | 已被 AGENT-r0 取代 |
| [`design/roadmap/ROADMAP-v2.md`](design/roadmap/ROADMAP-v2.md) | 已被 DEV-PLAN-r0 取代 |

## 历史文档（仅作参考，未必反映现行决策）

`docs/` 与 `docs/zh/` 下的 11 篇文档保留，但其中：
- `09-mbclaw-full-vision.md` 的 13 项目方案 → 已被新 MVP 裁剪；
- `11-implementation-status.md` 的"33/34 完成"声明 → 已被审计修正（见 audit）。

如需复用旧文档结论，先对照 `design/audit/`。
