# 架构 v2 — 收敛后的 MBclaw

**版本**: v2
**日期**: 2026-06-21
**适用**: R0-R1

---

## 1. 一图概览

```
┌────────────────────────────────────────────────────────┐
│  调用方                                                  │
│  CLI / OpenHands / 任何 HTTP 客户端                      │
└────────────┬───────────────────────────────────────────┘
             │ HTTP REST
             ▼
┌────────────────────────────────────────────────────────┐
│  FastAPI（单进程）                                        │
│  Routers（8 个）：                                        │
│    health / users / projects / sessions / messages       │
│    memory / search / dna                                 │
└────────────┬───────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────┐
│  Services（7 个，单一职责）                                │
│  ┌──────────────┬──────────────┬──────────────────────┐│
│  │transcript    │summary       │keyword               ││
│  │（M1 写入）    │（M2 摘要）    │（M3 关键词）           ││
│  ├──────────────┼──────────────┼──────────────────────┤│
│  │search        │dna           │memory_store          ││
│  │（M4 检索）    │（M5 DNA）     │（M6 双态记忆）         ││
│  ├──────────────┴──────────────┴──────────────────────┤│
│  │llm_client（统一 LLM 入口，所有调用走这里）            ││
│  └─────────────────────────────────────────────────────┘│
└────────────┬───────────────────────────────────────────┘
             │
   ┌─────────┴──────────┐
   ▼                    ▼
┌─────────┐      ┌──────────────┐
│ SQLite  │      │  文件系统     │
│ (WAL)   │      │  transcripts/│
│ + FTS5  │      │  memory/     │
└─────────┘      └──────────────┘
```

---

## 2. 7 个服务的职责边界

| Service | 职责（独占） | 不做 |
|---|---|---|
| **transcript** | 消息持久化（DB+JSONL）+ 写锁 | 不做摘要、不做检索 |
| **summary** | 会话结束触发 LLM 摘要 | 不做关键词、不做分类 |
| **keyword** | jieba 分词 + TF-IDF | 不做语义、不做向量 |
| **search** | SQLite FTS5 + 关键词查询 | 不做向量、不做分层（MVP） |
| **dna** | Project DNA 增量更新 | 不做心理画像 |
| **memory_store** | MEMORY.md 双态读写 | 不做审批、不做归档 |
| **llm_client** | 统一 LLM 调用 + provider 适配 | 业务不直连 LLM |

> **强制规则**：任何 service 想直接 `import openai/httpx` 调 LLM —— **不允许**，必须走 `llm_client`。

---

## 3. 关键不变量（架构红线）

1. **单一 LLM 入口**：业务代码不允许出现 provider 细节。
2. **不引入新存储**（R1）：不加 Redis / Postgres / 向量库。
3. **不引入消息队列**：MVP 阶段所有"异步"都是 BackgroundTasks 或简单 asyncio。
4. **Service 之间不互相 import service**：避免环依赖；共享逻辑下沉到 utils 或上调到 router 协调。
5. **Router 不写业务逻辑**：只做参数校验 + service 调用 + 响应序列化。

---

## 4. 与现有 Lite 代码的迁移映射

| 保留（重构） | 移出 Core（归档 Design 或 Memory） |
|---|---|
| `database.py` `config.py` `main.py` | `psychology_engine.py` → Memory |
| `services/memory_store.py` | `services/utopia_service.py` → Memory |
| `services/transcript_service.py` | `services/chat_extractor.py` → Memory |
| `services/summary_service.py` | `services/collision_engine.py` → Memory |
| `services/keyword_service.py` | `services/sub_agent_coordinator.py` → Memory |
| `services/search_service.py`（裁剪掉 L2/L3） | `services/dual_key.py` → Memory |
| `services/dna_service.py` | `services/skill_extractor.py` → Design（R2） |
| 新建 `services/llm_client.py`（统一现有散落入口） | `services/agent_runtime.py` → Design（R2） |
| | `services/integration_service.py` (11 平台) → Design（R3） |
| | `services/approval_gate.py`（多维评分） → 简化版进 memory_store |
| | `services/curator.py` → Design（R2） |
| | `services/classification_service.py`（树状） → Design（R2） |
| | `services/idle_scheduler.py` → 简化为定时器，并入 main |
| | `services/i18n_service.py` → Design（R3） |
| | `services/snapshot_service.py` → 简化为 SQLite VACUUM INTO，并入 utils |
| | `services/startup_checker.py` → 简化为 5 行 init |
| | `services/tool_service.py` / `model_service.py` → Design（R2） |
| | `services/layered_search.py` → 删（合并入 search） |
| | `services/auto_mode.py` / `task_queue.py` / `message_priority.py` → Design（R2） |
| | `services/feedback_service.py` → Design（R2） |
| | `services/vector_store.py` → Design（R2） |

---

## 5. 部署形态

- **R1**：单进程 `uvicorn app.main:app`，SQLite 文件，`data/` 目录。
- **R1 不部署**：Docker / K8s / 多副本。
- **R3 再上**：Docker 镜像（K8s manifests 移走，保留作 R3 参考）。

---

## 6. 可观测性（最小）

- 结构化日志（stdlib `logging` + JSON formatter）。
- 一个 `/health` 端点，返回：DB 可写 / 磁盘可用 / 最近一次 LLM 调用状态。
- **不上** Prometheus / OpenTelemetry（R3 再说）。

---

## 7. 给 OpenHands 的实施指令格式

每个迁移任务必须满足：
```
- 输入：Core 当前的 services/X.py
- 输出：删除 / 简化 / 保留 三选一
- 验证：变更后 pytest 通过
- 边界：仅修改文件清单内的文件
```
否则 OpenHands 拿不到可执行任务。
