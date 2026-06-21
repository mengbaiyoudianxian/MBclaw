# MBclaw R0 系统架构

**版本**: r0
**日期**: 2026-06-21
**取代**: `design/architecture/ARCH-v2.md`

---

## 1. 模块划分（7 文件）

```
mbclaw-lite/r0/
├── app/
│   ├── main.py       # FastAPI 入口 + 路由注册 + init
│   ├── db.py         # SQLite 连接 + schema + FTS 触发器
│   ├── models.py     # SQLAlchemy 模型（5 表）
│   ├── llm.py        # 统一 LLM 客户端（唯一 provider 出口）
│   ├── memory.py     # MemoryRepo 抽象（唯一记忆操作入口）
│   ├── pipeline.py   # close_session 同步管线
│   └── api.py        # 5 端点
├── tests/{unit, e2e}/
├── data/             # gitignore
└── requirements.txt
```

预算：≤1500 行（不含测试）。

### 模块边界
| 模块 | 唯一职责 | 不做 |
|---|---|---|
| main | 装配 | 业务 |
| db | 连接/schema/事务 | 业务查询 |
| models | ORM 定义 | 业务方法 |
| llm | 唯一 provider 出口 | 知道业务 |
| memory | 唯一记忆抽象层 | 知道 HTTP |
| pipeline | 沉淀编排 | 知道 HTTP |
| api | HTTP 协议适配 | 业务 |

调用方向：`api → pipeline → memory → db/llm`，反向禁止。

---

## 2. 数据流

### 流 A 写入
```
POST /sessions/{id}/messages
  → api → db.add(Message) → messages_fts(trigger) → JSONL
```
同步 <50ms。

### 流 B 沉淀
```
POST /sessions/{id}/close
  → pipeline.close(sid):
      load → llm.summarize → memory.write_session_memory → status=closed
```
同步 5-60s。

### 流 C 注入
```
POST /sessions
  → memory.render_injection_for_new_session():
      self-prime query → query() → render → 写入 messages(system)
```
同步 <500ms。

---

## 3. 记忆系统

详见 `design/memory/MEMORY-SYSTEM-r0.md`。

核心：**唯一 MemoryRepo 抽象**，3 个方法：
- `write_session_memory(sid, summary, keywords, experiences)`
- `query(q, top_n) -> list[Hit]`
- `render_injection_for_new_session(exclude_sid) -> str | None`

不变量：
1. 业务只通过 MemoryRepo 操作记忆（CI grep 强制）
2. MemoryRepo 不调 LLM（CI grep 强制）
3. 注入 ≤800 字符（单测兜底）

---

## 4. API 设计

| 方法 | 路径 | 用途 | SLA |
|---|---|---|---|
| POST | /sessions | 开会话 + 自动注入 | 500ms |
| POST | /sessions/{id}/messages | 写消息 | 50ms |
| POST | /sessions/{id}/close | 关会话沉淀 | 60s |
| GET | /sessions/{id}/messages | 读对话 | 100ms |
| GET | /search?q=... | 检索 | 200ms |

详细见 `design/mvp/MVP-r0-1week.md` §3。

---

## 5. 存储

### 5.1 单一 SQLite
```
data/
├── mbclaw.db                  # WAL 模式
└── transcripts/{sid}.jsonl    # 事故兜底备份
```

无 Postgres/Redis/ChromaDB/Qdrant/ES/对象存储/消息队列。

### 5.2 表
5 业务表 + 2 FTS。详见 `design/memory/MEMORY-SYSTEM-r0.md` §1。

### 5.3 PRAGMA
```sql
PRAGMA journal_mode=WAL;
PRAGMA synchronous=NORMAL;
PRAGMA cache_size=-20000;
PRAGMA temp_store=MEMORY;
```

### 5.4 增长估算
| 项 | 100 sessions | 1 万 sessions |
|---|---|---|
| messages | ~10 MB | ~1 GB |
| FTS index | ~15 MB | ~1.5 GB |

1 万 sessions 内 SQLite 不会瓶颈。

---

## 6. 是否需要 Agent

**R0 不需要。明确判断。**

| 维度 | 判断 |
|---|---|
| Agent 是 MBclaw 自身的一部分？ | ❌ 不是 |
| Agent 是 MBclaw 的消费者？ | ✅ 是 |
| 当前需要 Agent 自动写记忆？ | ❌ HTTP 调用方触发即可 |
| 当前需要 Agent 自动检索？ | ❌ C5 注入已覆盖 |

### 角色定位
```
外部 Agent（OpenHands/Claude Code/LangGraph/自写 App）
        │ HTTP REST
        ▼
    MBclaw 记忆服务
```

### Lite 中 Agent 代码处置
| 模块 | 处置 |
|---|---|
| agent_runtime.py (438) | Memory drafts/legacy/agent/，R2 重写参考 |
| sub_agent_coordinator | Memory（永久移出） |
| dual_key | Memory（永久移出） |
| auto_mode | Memory（永久移出） |
| approval_gate | R0 简化为单开关 |
| task_queue / message_priority | Memory（无并发） |

### R2 启用 Agent 触发条件
1. 用户明确说"我希望 MBclaw 代我执行 xxx"
2. 场景具体
3. 现有 Agent 框架不能满足

任一不满足 → 继续不做。

---

## 7. 技术选型理由

| 技术 | 选 | 为什么 | 为什么不选其他 |
|---|---|---|---|
| 语言 | Python 3.11+ | LLM 生态 + jieba + FastAPI | Go 缺生态；Rust 慢；Node 部署逊 |
| 框架 | FastAPI | 类型+OpenAPI+异步+轻 | Flask 无类型；Django 太重 |
| ORM | SQLAlchemy 2.0 | SQLite/PG 兼容（R2 迁移不卡） | Tortoise 生态薄；Peewee 弱 |
| DB | SQLite WAL+FTS5 | 单文件零运维 | Postgres 运维浪费；Mongo 关系建模难 |
| 检索 | FTS5 + jieba | 中文分词+索引一体 | ES 杀牛刀；Meilisearch 多进程 |
| 关键词 | jieba + TF-IDF | 中文 NLP 标准 | spaCy 英文；hanlp 依赖重 |
| 向量库 | ❌ 不上 | 限制 + 无数据 | R2 再说 |
| LLM | httpx + OpenAI 兼容 | provider 通用 | langchain 太重；llama-index 偏 RAG |
| 调度 | ❌ 不要 | C2 同步够 | Celery/RQ MVP 无需 |
| 缓存 | ❌ 不要 | SQLite 自带 | Redis 浪费 |
| 部署 | uvicorn 一行 | 简单 | Docker 可选；K8s R3 |
| 测试 | pytest + TestClient | 标准 | unittest 缺 fixture |
| 日志 | logging + JSON | 零依赖 | structlog/loguru 多余 |
| 依赖 | requirements.txt pin | CI 友好 | poetry/uv R1 |
| 配置 | .env 5 变量 | 简单 | pydantic-settings R1 |

**一句话**：能用标准库就用，能 SQLite 就 SQLite，能同步就同步，能不要就不要。

---

## 与 ARCH-v2 差异

| 项 | ARCH-v2 | ARCH-r0 |
|---|---|---|
| Service | 7 | 6（合并 dna/audit 入 memory） |
| Router | 8 | 1（单文件） |
| 表 | 8 | 5（去 users/projects/dna/audit） |
| MemoryRepo | 隐含 | 强制 |
| Agent | R2 启用 | 明确不是 |

---

## Core / Design 分流

### Core（R0 落地）
- FastAPI 单进程 / SQLite+FTS5 / 5 业务表 / 统一 llm.py / MemoryRepo / pipeline / 5 端点 / 端到端测试

### Design（信号触发）
- MemoryRepo 双态扩展（C5 命中率 < 60%）
- 向量检索（FTS5 召回 < 70%）
- Project DNA（多项目混淆）
- Agent Runtime 重写（用户具体场景）
- 多模型调度（≥3 provider）
- Reflection/Dreaming（摘要 < 75%）
- 11 平台网关（单平台 DAU > 50）
- K8s（单机扛不住）
