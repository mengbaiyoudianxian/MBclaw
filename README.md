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

| 文档 | 用途 |
|---|---|
| [`design/audit/AUDIT-2026-06-21.md`](design/audit/AUDIT-2026-06-21.md) | 现状审计（10379 行 Lite 的真实评估） |
| [`design/mvp/MVP-v2.md`](design/mvp/MVP-v2.md) | 重新定义的 MVP 边界（**裁剪 70%**） |
| [`design/architecture/ARCH-v2.md`](design/architecture/ARCH-v2.md) | 收敛后架构（7 服务，非 39 服务） |
| [`design/database/SCHEMA-v2.md`](design/database/SCHEMA-v2.md) | 核心数据模型（8 张表，非 24 张） |
| [`design/agent/AGENT-v2.md`](design/agent/AGENT-v2.md) | Agent 设计（单循环 + 工具子集） |
| [`design/roadmap/ROADMAP-v2.md`](design/roadmap/ROADMAP-v2.md) | R0→R3 路线图 |

## 历史文档（仅作参考，未必反映现行决策）

`docs/` 与 `docs/zh/` 下的 11 篇文档保留，但其中：
- `09-mbclaw-full-vision.md` 的 13 项目方案 → 已被新 MVP 裁剪；
- `11-implementation-status.md` 的"33/34 完成"声明 → 已被审计修正（见 audit）。

如需复用旧文档结论，先对照 `design/audit/`。
