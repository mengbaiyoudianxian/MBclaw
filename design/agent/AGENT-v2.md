# Agent 设计 v2

**版本**: v2
**日期**: 2026-06-21
**状态**: R2 启用（R0-R1 不实装 Agent，只提供"记忆"API 供外部 Agent 调用）

---

## 1. 重要立场

**R1 阶段 MBclaw 不是 Agent**，是 **"长期记忆服务"**。
Agent（OpenHands / OpenClaw / Claude Code / 你自己写的）是消费者，通过 HTTP 调用 MBclaw 的记忆 API。

> 不要在 MVP 阶段做 Agent Runtime。当前的 `agent_runtime.py`（438 行）属于先造引擎再找车——R1 移出。

---

## 2. R2 才启用的 Agent 形态

### 2.1 单循环（不是多 agent）
```
while not done and step < max_steps:
    context = build_context(memory, dna, tools)
    plan = llm.plan(context, user_msg)
    if plan.tool_call:
        result = run_tool(plan.tool, plan.args)
        memory.log(result)
    else:
        return plan.final_answer
```

不做：
- 多 sub-agent 并行
- Dual-Key 互评
- 思维碰撞
- 心理画像驱动

理由：MVP 阶段我们没有任何证据证明这些复杂结构带来收益。

### 2.2 工具集（R2 启动版，最多 5 个）
| 工具 | 输入 | 输出 | 安全级 |
|---|---|---|---|
| `memory.search` | query | top-N 片段 | 只读 |
| `memory.write` | op + content | 审计后落库 | 写（走 memory_audit） |
| `dna.update` | field + value | 增量更新 | 写 |
| `transcript.read` | session_id | JSONL | 只读 |
| `shell.run`（可选） | cmd | stdout | **白名单沙盒**，默认关 |

### 2.3 上下文构造
```
[system] 项目 DNA + 最近 5 条相关 memory + 当前会话最近 N 轮
[user]   最新消息
```

不做：分层注入、动态预算、嵌套 prompt——MVP 阶段没数据支撑。

---

## 3. 安全（最小集）

- 任何 `memory.write` / `dna.update` 写入 → `memory_audit` 留痕。
- 没有"风险评分多维度"——MVP 用单一开关：`AUTO_APPROVE_WRITE = true/false`。
- `shell.run` 默认禁用；启用必须在配置文件明确写白名单。

---

## 4. 失败与回退

- LLM 失败 → 重试 1 次 → 落库错误 → 返回 503。
- 工具失败 → 计入"failed_approaches"，自我学习的最小机制。
- 不做：自动重写 / 自动多方案。

---

## 5. 与现 Lite `agent_runtime.py` 的处理

- **R1**: 整文件从 Core 移出 → Design `design/agent/legacy/agent-runtime-v1.md`（仅留摘要）→ 代码到 Memory 仓库 `drafts/agent-runtime-v1/`。
- **R2 启动**：从零写一个 < 200 行的新实现，绝不复用旧的 438 行。
