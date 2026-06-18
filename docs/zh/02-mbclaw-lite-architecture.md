# 02 — MBclaw-Lite MVP 架构设计

## 总体架构

```
┌─────────────────────────────────────────────────────────┐
│                      外部调用方                           │
│          OpenHands / OpenClaw / LangGraph / CLI          │
└────────────┬────────────────────────────────────────────┘
             │  HTTP REST API
             ▼
┌─────────────────────────────────────────────────────────┐
│                    FastAPI 应用层                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │ Routers  │  │ Schemas  │  │ Services │  │ Models  │ │
│  │ (API 层) │  │ (校验层)  │  │ (业务层)  │  │ (ORM 层)│ │
│  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │
└────────────┬─────────────────────────────────────────────┘
             │  SQLAlchemy
             ▼
┌─────────────────────────────────────────────────────────┐
│                    SQLite 数据库                          │
│   users → projects → sessions → messages                 │
│                    ├→ summaries                          │
│                    ├→ keywords                           │
│                    └→ project_dna                        │
└─────────────────────────────────────────────────────────┘
```

## 技术选型

| 组件 | 选择 | 原因 |
|------|------|------|
| 语言 | Python 3.10+ | 生态丰富，AI / Agent 领域标准 |
| 框架 | FastAPI | 高性能、自动文档、类型安全 |
| ORM | SQLAlchemy 2.0 | 成熟稳定，支持 SQLite |
| 数据库 | SQLite（WAL 模式） | 零配置、轻量、单文件 |
| 分词 | jieba | 中文分词标准库 |
| 存储 | JSON 文件 | 数据目录独立，便于备份 |

## 数据库 Schema（7 张表）

```
users
  id INTEGER PK, name TEXT UNIQUE, created_at TEXT

projects
  id INTEGER PK, user_id FK→users, name TEXT, description TEXT
  created_at TEXT, updated_at TEXT

sessions
  id INTEGER PK, project_id FK→projects, session_number INTEGER
  title TEXT, status TEXT（active | completed）
  started_at TEXT, ended_at TEXT

messages
  id INTEGER PK, session_id FK→sessions
  role TEXT（user | assistant | system）, content TEXT, created_at TEXT

summaries
  id INTEGER PK, session_id FK→sessions UNIQUE
  topic TEXT, conclusions TEXT, decisions TEXT, next_steps TEXT

keywords
  id INTEGER PK, session_id FK→sessions, project_id FK→projects
  keyword TEXT, weight REAL

project_dna
  id INTEGER PK, project_id FK→projects UNIQUE
  goals JSON, successful_approaches JSON, failed_approaches JSON
  tools JSON, models JSON, next_plans JSON
```

## Phase 1 核心功能

### 1. 保存所有对话
- POST 消息到活跃 Session
- 保存原始内容、时间、项目名称、会话编号

### 2. 自动生成总结
- Session complete 时触发
- 提取：主题、核心结论、关键决定、下一步计划
- 基于关键词规则匹配（可后续升级为 LLM 调用）

### 3. 自动提取关键词
- jieba 分词 + TF-IDF 权重计算
- 中英文混合支持
- 去停用词

### 4. 项目分类
- 一个用户多个项目
- 每个项目独立的 sessions / messages / summaries / keywords / dna

### 5. Project DNA
- 每个项目一个 project_dna.json
- 增量更新（不覆盖已有内容）
- 每次 session complete 时自动合并 decisions / next_steps

### 6. 搜索历史
- 全文搜索：项目名、会话标题、消息内容、总结、关键词
- 支持按 project_id 过滤

## Phase 2 — OpenClaw 参考增强（已完成）

### 三层记忆模型
- **MEMORY.md** — 持久精选事实，会话启动时加载
- **YYYY-MM-DD.md** — 每日工作笔记（今天 + 昨天自动加载）
- **DREAMS.md** — 后台整合日记

### Memory Flush（记忆冲刷）
- 会话完成时自动保存上下文到 daily notes
- 包含：主题、结论、决定、下一步、关键词、最近对话

### Dreaming（梦想整合）
- 后台整合最近 7 天的总结和关键词
- 评分 → 筛选 → 提升到 MEMORY.md

### JSONL 会话记录
- `data/transcripts/{session_id}.jsonl` — 只追加审计日志
- 写锁保护竞态条件

## API 端点

| 方法 | 路径 | 阶段 | 说明 |
|------|------|------|------|
| POST/GET | /api/users | P1 | 创建 / 列出用户 |
| POST/GET | /api/projects | P1 | 创建 / 列出项目 |
| GET | /api/projects/{id} | P1 | 获取项目详情 |
| POST/GET | /api/projects/{id}/sessions | P1 | 创建 / 列出会话 |
| PATCH | /api/projects/{id}/sessions/{id}/complete | P1 | 完成会话（触发总结+关键词+DNA+Flush+Transcript） |
| POST/GET | /api/sessions/{id}/messages | P1 | 添加 / 列出消息 |
| GET/POST | /api/sessions/{id}/summary | P1 | 获取 / 重新生成总结 |
| GET | /api/projects/{id}/keywords | P1 | 获取项目关键词 |
| GET/PATCH | /api/projects/{id}/dna | P1 | 获取 / 更新项目 DNA |
| GET | /api/search?q=xxx | P1 | 全文搜索 |
| GET/PUT | /api/projects/{id}/memory/durable | P2 | MEMORY.md 读写 |
| GET/POST | /api/projects/{id}/memory/daily | P2 | 每日笔记读写 |
| GET | /api/projects/{id}/memory/dreams | P2 | 查看梦想日记 |
| POST | /api/projects/{id}/memory/dream | P2 | 执行记忆整合 |

## 文件夹结构

```
MBclaw-Lite/
├── app/
│   ├── main.py              # FastAPI 入口
│   ├── config.py            # 配置
│   ├── database.py          # SQLite + 建表
│   ├── models/              # 7 个 ORM 模型
│   ├── schemas/             # 9 个 Pydantic 模型
│   ├── routers/             # 9 个路由模块（含 memory）
│   └── services/            # 6 个业务服务（含 memory + transcript）
├── data/mbclaw.db           # SQLite 数据库
├── data/transcripts/        # JSONL 会话记录
├── data/memory/             # 三层记忆文件
├── tests/                   # 18 个测试
└── requirements.txt
```

## 扩展预留点

- `app/services/` 下可加 `agent_service.py`（Agent 执行逻辑）
- `app/services/llm/` 下可放不同 LLM 适配器
- `app/routers/` 下可加 `webhooks.py`（接收 OpenHands 回调）
- 数据模型已预留 `user_id` 用于多用户扩展
