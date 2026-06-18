# 10 — MBclaw 长期记忆系统制造蓝图

> 完整制造规范。不实施，只详细列出。
> 覆盖全部 13 项目所需的所有组件、数据模型、API、数据流、文件结构。

---

## 一、总体系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        接入层                                    │
│  OpenHands  │  OpenClaw Plugin  │  CLI  │  Android miclaw APK   │
└────────────┬──────────┬─────────┬───────┬───────────────────────┘
             │          │         │       │
             ▼          ▼         ▼       ▼
┌─────────────────────────────────────────────────────────────────┐
│                     MBclaw Gateway (FastAPI)                     │
│                                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │
│  │ Task     │ │ Message  │ │ Model    │ │ Provider         │  │
│  │ Queue    │ │ Priority │ │ Scheduler│ │ Adapters         │  │
│  │ (项目七) │ │ (项目七) │ │ (项目十二)│ │ (项目十三:MiMo)  │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Memory Core                             │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐ │  │
│  │  │ Classify │ │ Search   │ │ Dream    │ │ Snapshot     │ │  │
│  │  │ Engine   │ │ Engine   │ │ Engine   │ │ Manager      │ │  │
│  │  │(项目二)  │ │(项目六)  │ │(已有)    │ │(项目三)      │ │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └─────────────┘ │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐ │  │
│  │  │ Tool     │ │ Key Mgr  │ │ Collab   │ │ Auto Decision│ │  │
│  │  │ Index    │ │ (项目五) │ │ Channel  │ │ Engine       │ │  │
│  │  │(项目十一)│ │          │ │(项目十)  │ │(项目四)      │ │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └─────────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────┬────────────────────────────────────────────────────┘
             │
    ┌────────┴────────┬──────────────┬──────────────┐
    ▼                 ▼              ▼              ▼
┌────────┐    ┌────────────┐ ┌────────────┐ ┌──────────────┐
│ SQLite │    │ Memory     │ │Transcripts │ │ Snapshots    │
│ (8表)  │    │ Files (.md)│ │ (.jsonl)   │ │ (git + .db)  │
└────────┘    └────────────┘ └────────────┘ └──────────────┘
```

---

## 二、完整数据库 Schema（8 张表）

### 2.1 users（已有）
```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 2.2 projects（已有）
```sql
CREATE TABLE projects (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL REFERENCES users(id),
    name TEXT NOT NULL,
    description TEXT DEFAULT '',
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 2.3 sessions（已有）
```sql
CREATE TABLE sessions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_id INTEGER NOT NULL REFERENCES projects(id),
    session_number INTEGER NOT NULL,
    title TEXT DEFAULT '',
    status TEXT NOT NULL DEFAULT 'active' CHECK(status IN ('active','completed','interrupted')),
    started_at TEXT NOT NULL DEFAULT (datetime('now')),
    ended_at TEXT
);
-- interrupted 状态是项目七新增：被打断但未完成的会话
```

### 2.4 messages（已有，需扩展）
```sql
CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id INTEGER NOT NULL REFERENCES sessions(id),
    role TEXT NOT NULL CHECK(role IN ('user','assistant','system')),
    content TEXT NOT NULL,
    thinking TEXT DEFAULT '',           -- 项目一新增：AI 思考过程
    message_type TEXT DEFAULT 'message'  -- 项目一新增：message|code_change|decision
        CHECK(message_type IN ('message','code_change','thinking','decision')),
    metadata TEXT DEFAULT '{}',          -- 项目一新增：JSON，存 code_change 的 file_path/diff
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 2.5 summaries（已有）
```sql
CREATE TABLE summaries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id INTEGER UNIQUE NOT NULL REFERENCES sessions(id),
    topic TEXT DEFAULT '',
    conclusions TEXT DEFAULT '',
    decisions TEXT DEFAULT '',
    next_steps TEXT DEFAULT '',
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 2.6 keywords（已有）
```sql
CREATE TABLE keywords (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id INTEGER NOT NULL REFERENCES sessions(id),
    project_id INTEGER NOT NULL REFERENCES projects(id),
    keyword TEXT NOT NULL,
    weight REAL NOT NULL DEFAULT 1.0
);
CREATE INDEX idx_keywords_project ON keywords(project_id);
CREATE INDEX idx_keywords_keyword ON keywords(keyword);
```

### 2.7 project_dna（已有）
```sql
CREATE TABLE project_dna (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_id INTEGER UNIQUE NOT NULL REFERENCES projects(id),
    goals TEXT DEFAULT '[]',              -- JSON array
    successful_approaches TEXT DEFAULT '[]',
    failed_approaches TEXT DEFAULT '[]',
    failed_approaches_detail TEXT DEFAULT '[]',  -- 项目二新增：结构化失败详情
    tools TEXT DEFAULT '[]',
    models TEXT DEFAULT '[]',
    next_plans TEXT DEFAULT '[]',
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 2.8 action_memories（已有）
```sql
CREATE TABLE action_memories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id INTEGER REFERENCES sessions(id),
    project_id INTEGER NOT NULL REFERENCES projects(id),
    action TEXT NOT NULL,
    permissions TEXT DEFAULT '',
    timing TEXT DEFAULT '',
    expiry TEXT DEFAULT '',
    source_authority TEXT DEFAULT 'medium'
);
```

### 2.9 topic_tree（项目二新增）
```sql
CREATE TABLE topic_tree (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_id INTEGER NOT NULL REFERENCES projects(id),
    parent_id INTEGER REFERENCES topic_tree(id),  -- 树状结构
    name TEXT NOT NULL,                   -- 节点名称（如 "数据库设计"）
    node_type TEXT NOT NULL DEFAULT 'topic'  -- topic|summary|failed_detail
        CHECK(node_type IN ('topic','summary','session_ref','failed_detail')),
    summary TEXT DEFAULT '',              -- 粗略总结（≤200字）
    detail TEXT DEFAULT '',               -- 详细内容
    session_refs TEXT DEFAULT '[]',       -- 关联的 session IDs (JSON array)
    keyword_refs TEXT DEFAULT '[]',       -- 关联的关键词 IDs (JSON array)
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_topic_tree_project ON topic_tree(project_id);
CREATE INDEX idx_topic_tree_parent ON topic_tree(parent_id);
```

### 2.10 keyword_index（项目二新增 — 反向索引）
```sql
CREATE TABLE keyword_index (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    keyword TEXT NOT NULL,
    project_id INTEGER NOT NULL REFERENCES projects(id),
    session_ids TEXT NOT NULL DEFAULT '[]',  -- JSON array of session IDs
    topic_node_ids TEXT NOT NULL DEFAULT '[]', -- JSON array of topic_tree IDs
    weight REAL NOT NULL DEFAULT 1.0,
    updated_at TEXT NOT NULL DEFAULT (datetime('now')),
    UNIQUE(keyword, project_id)
);
CREATE INDEX idx_keyword_index_keyword ON keyword_index(keyword);
```

### 2.11 tools（项目十一新增）
```sql
CREATE TABLE tools (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE NOT NULL,
    summary TEXT NOT NULL,                -- ≤100 字摘要
    tags TEXT NOT NULL DEFAULT '[]',      -- JSON array of tags
    full_description TEXT NOT NULL,       -- 完整文档
    embedding TEXT DEFAULT '',            -- 向量序列化（JSON array of floats）
    usage_categories TEXT DEFAULT '[]',   -- 适用 topic 类型 (JSON array)
    usage_count INTEGER DEFAULT 0,
    success_count INTEGER DEFAULT 0,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 2.12 model_profiles（项目十二新增）
```sql
CREATE TABLE model_profiles (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,                   -- "MiMo", "GPT-4o", "Claude 4"
    provider TEXT NOT NULL,               -- "mimo", "openai", "anthropic"
    api_key_ref TEXT NOT NULL,            -- 对应用户的哪个 key
    capabilities TEXT NOT NULL DEFAULT '{}', -- {"reasoning":0.9, "coding":0.85, ...}
    cost_per_1k_input REAL DEFAULT 0,
    cost_per_1k_output REAL DEFAULT 0,
    max_tokens INTEGER DEFAULT 4096,
    tool_compatibility TEXT DEFAULT '{}', -- {"browser":0.8, "file_edit":0.95, ...}
    is_available INTEGER DEFAULT 1,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 2.13 shared_channel（项目十新增）
```sql
CREATE TABLE shared_channel (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_id INTEGER NOT NULL REFERENCES projects(id),
    agent_id TEXT NOT NULL,               -- 子对话标识
    task TEXT NOT NULL,                   -- 任务描述
    status TEXT NOT NULL DEFAULT 'completed' CHECK(status IN ('completed','failed','in_progress')),
    findings TEXT DEFAULT '[]',           -- JSON array
    problems TEXT DEFAULT '[]',           -- JSON array
    solutions TEXT DEFAULT '[]',          -- JSON array
    reusable TEXT DEFAULT '[]',           -- JSON array of reusable file paths
    conflicts TEXT DEFAULT '[]',          -- JSON array of conflict descriptions
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_shared_channel_project ON shared_channel(project_id);
```

### 2.14 task_queue（项目七新增）
```sql
CREATE TABLE task_queue (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_id INTEGER NOT NULL REFERENCES projects(id),
    session_id INTEGER REFERENCES sessions(id),
    task_type TEXT NOT NULL DEFAULT 'user_request',
    status TEXT NOT NULL DEFAULT 'queued'
        CHECK(status IN ('queued','running','paused','completed','failed')),
    priority INTEGER NOT NULL DEFAULT 0,  -- 越大越优先，用户新消息=100
    payload TEXT NOT NULL DEFAULT '{}',   -- JSON: task content, context, state
    checkpoint TEXT DEFAULT '',           -- 项目七：被打断时的状态快照 (JSON)
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 2.15 snapshots（项目三新增）
```sql
CREATE TABLE snapshots (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_id INTEGER NOT NULL REFERENCES projects(id),
    session_id INTEGER REFERENCES sessions(id),
    tag TEXT NOT NULL,                    -- git tag 名称
    trigger_reason TEXT NOT NULL,         -- "dna_change"|"user_mark"|"keyword_match"
    db_backup_path TEXT,                  -- 数据库热备份路径
    git_commit_hash TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

---

## 三、完整 API 端点清单（30 个）

### Phase 1 已有（13 个）
```
POST   /api/users                             创建用户
GET    /api/users                             列出用户
POST   /api/projects                          创建项目
GET    /api/projects                          列出项目
GET    /api/projects/{id}                     获取项目
POST   /api/projects/{id}/sessions            创建会话
GET    /api/projects/{id}/sessions            列出会话
PATCH  /api/projects/{id}/sessions/{id}/complete  完成会话(触发全流程)
POST   /api/sessions/{id}/messages            添加消息
GET    /api/sessions/{id}/messages            列出消息
GET    /api/sessions/{id}/summary             获取总结
POST   /api/sessions/{id}/summary             重新生成总结
GET    /api/projects/{id}/keywords            获取关键词
GET    /api/projects/{id}/dna                 获取 DNA
PATCH  /api/projects/{id}/dna                 更新 DNA
GET    /api/search                            全文搜索
```

### Phase 2 已有（5 个）
```
GET    /api/projects/{id}/memory/durable      MEMORY.md 读取
PUT    /api/projects/{id}/memory/durable      MEMORY.md 写入
GET    /api/projects/{id}/memory/daily        每日笔记读取
POST   /api/projects/{id}/memory/daily        每日笔记追加
GET    /api/projects/{id}/memory/dreams       DREAMS.md 读取
POST   /api/projects/{id}/memory/dream        执行 dreaming 整合
GET    /api/projects/{id}/actions             列出 action_memories
POST   /api/projects/{id}/actions             手动创建 action_memory
```

### Phase 3 新增（12 个）
```
# 项目二 — 树状分类
GET    /api/projects/{id}/topics              获取话题树
POST   /api/projects/{id}/topics/classify     触发分类整理
GET    /api/projects/{id}/topics/{node_id}    获取话题节点详情
GET    /api/projects/{id}/topics/failed       获取所有失败方案

# 项目六 — 分层搜索
POST   /api/projects/{id}/search/semantic     语义搜索(L3: embedding)
GET    /api/projects/{id}/search/prefetch     预调用：获取当前对话的相关记忆

# 项目一 — 增强日志
GET    /api/sessions/{id}/transcript          获取增强 transcript(含 thinking+code_change)
GET    /api/sessions/{id}/transcript/export   导出完整 transcript 文件

# 项目三 — 快照
GET    /api/projects/{id}/snapshots           列出快照
POST   /api/projects/{id}/snapshots           手动创建快照
GET    /api/projects/{id}/snapshots/{id}      获取快照详情
POST   /api/projects/{id}/snapshots/{id}/restore  恢复到快照

# 项目四 — 自动决策
POST   /api/projects/{id}/auto/decide         对当前选择自动决策（返回推荐方案）

# 项目五 — 双 Key 协作
POST   /api/projects/{id}/collab/review       用 Key2 评审最近产出

# 项目七 — 任务队列
GET    /api/tasks                             列出所有任务状态
POST   /api/tasks/{id}/pause                  暂停任务
POST   /api/tasks/{id}/resume                 恢复任务
POST   /api/tasks/{id}/cancel                 取消任务

# 项目十 — 子对话协同
GET    /api/projects/{id}/channel             获取共享通道内容
POST   /api/projects/{id}/channel             发布到共享通道
POST   /api/projects/{id}/channel/dedup       检查任务是否已有结果

# 项目十一 — 工具管理
GET    /api/tools                             列出所有工具(L1 摘要)
POST   /api/tools                             注册新工具
GET    /api/tools/{id}                        获取工具详情(L3 完整描述)
POST   /api/tools/search                      向量语义搜索工具
GET    /api/projects/{id}/tools/relevant      获取当前项目相关的工具

# 项目十二 — 模型管理
GET    /api/models                            列出所有模型(含能力评分)
POST   /api/models                            注册新模型
POST   /api/models/{id}/evaluate              重新评估模型能力(web search)
POST   /api/projects/{id}/models/select       为当前任务自动选择最优模型

# 项目十三 — MiMo 集成
POST   /api/providers/mimo/test               测试 MiMo 连接
GET    /api/providers/mimo/status             获取 MiMo 试用状态
POST   /api/projects/{id}/check/regression    检查最新修改是否回滚

# 项目八 — 中文化（所有端点返回中文错误消息）
# 无新增端点，改造所有现有 router 的错误消息
```

---

## 四、数据流详解

### 4.1 Session Complete 全流程（项目一+二+三触发点）

```
PATCH /api/projects/{id}/sessions/{id}/complete
  │
  ├─ 1. generate_summary(session)           # Phase 1: 关键词规则匹配
  ├─ 2. extract_keywords(session)           # Phase 1: jieba + TF-IDF
  ├─ 3. update_dna_from_session(session)    # Phase 1: DNA 增量合并
  ├─ 4. memory_flush(session)               # Phase 2: 写入 daily note
  ├─ 5. write_final_transcript(session)     # Phase 2: 写入 JSONL
  ├─ 6. extract_action_memories(session)    # Phase 2: 提取行动约束
  ├─ 7. classify_session(session)           # 项目二: 加入话题树
  ├─ 8. update_keyword_index(session)       # 项目二: 更新反向索引
  ├─ 9. check_breakthrough(project)         # 项目三: 检测突破
  │     └─ 如果 DNA 的 successful_approaches 变化
  │         ├─ git commit + tag
  │         ├─ SQLite .backup → snapshots/
  │         └─ INSERT INTO snapshots
  └─ 10. publish_to_shared_channel(session) # 项目十: 反思发布
```

### 4.2 实时记忆预调用流程（项目六）

```
用户发送消息
  │
  ├─ 1. 提取消息中的关键词 (jieba, <10ms)
  ├─ 2. L1: 关键词反向索引查找 (keyword_index 表, <10ms)
  │     └─ 返回直接匹配的 session_ids + topic_node_ids
  ├─ 3. L2: TF-IDF 相似度匹配 (与现有 summaries 比对, <100ms)
  │     └─ 返回相似度 > 阈值的 summaries
  ├─ 4. 判断：L1+L2 结果是否够用？
  │     ├─ 够用 → 注入 [相关记忆] 上下文块 → 继续对话
  │     └─ 不够 → L3: 向量语义搜索 (embedding API, <500ms)
  │           └─ 注入 [深层相关记忆] 上下文块 → 继续对话
  └─ 5. Token 预算控制：注入总量 < max_context * 0.3
```

### 4.3 任务优先级队列流程（项目七）

```
用户新消息到达（当前 Task A 正在执行）
  │
  ├─ 1. 创建新 Task B，priority=100
  ├─ 2. 等待 Task A 当前 tool call 完成 (safe point)
  ├─ 3. 保存 Task A 状态 → task_queue.checkpoint (JSON)
  ├─ 4. Task A status → 'paused'
  ├─ 5. Task B status → 'running'
  │     └─ 执行 Task B
  ├─ 6. Task B 完成后：
  │     ├─ 自动恢复 Task A (restore from checkpoint)
  │     └─ 或等待用户指令
  └─ 7. 用户可随时查看/暂停/恢复/取消任何任务
```

### 4.4 双 Key 协作循环（项目五）

```
用户触发协作
  │
  ├─ 1. Key1(executor) 接收到评审请求
  │     模型: 从 model_profiles 中选最高 coding 能力
  │     产出: code/solution
  ├─ 2. Key2(reviewer) 评审
  │     模型: 从 model_profiles 中选最高 reasoning 能力（与 Key1 不同模型或不同 key）
  │     评审维度:
  │       - 代码质量 (0-10)
  │       - 逻辑正确性 (0-10)
  │       - 安全性 (0-10)
  │       - 完整性 (0-10)
  │     输出: 评分 + 问题列表 + 改进建议
  ├─ 3. 判断: 总分 ≥ threshold (默认 32/40)?
  │     ├─ YES → 接受，结束循环
  │     └─ NO → Key1 根据建议修改 → 回到步骤 1
  ├─ 4. 循环次数上限: 用户可配置 1-6 次
  └─ 5. 日志: 每轮评审记录到 action_memory
```

### 4.5 子对话协同流程（项目十）

```
Sub-agent 完成任务
  │
  ├─ 1. 自动反思 (LLM 自我评估)
  │     ┌─────────────────────────────────────┐
  │     │ 反思模板:                            │
  │     │ - 我做了什么？                       │
  │     │ - 有什么新发现？                     │
  │     │ - 遇到了什么问题？                   │
  │     │ - 怎么解决的？                       │
  │     │ - 有什么可以复用的？                 │
  │     │ - 与谁可能有冲突？                   │
  │     └─────────────────────────────────────┘
  ├─ 2. 发布到 shared_channel
  │     └─ INSERT INTO shared_channel (agent_id, task, findings, ...)
  └─ 3. 其他 Sub-agent 启动新任务前：
        ├─ 查询 shared_channel: 是否有相似任务？
        │     └─ 相似度 > 80% → 复用结果，不重复做
        ├─ 查询 shared_channel: 是否有冲突？
        │     └─ 有冲突 → LLM 调解: 选最优或合并
        └─ 查询 shared_channel: 有什么可复用的？
              └─ 直接引用 reusable 文件/代码
```

### 4.6 三层工具索引流程（项目十一）

```
工具注册时:
  ├─ 1. 用户/系统注册工具: name + summary(≤100字) + full_description
  ├─ 2. 自动生成标签 (LLM 从 summary 提取)
  ├─ 3. 自动生成 embedding (调用 embedding API)
  ├─ 4. 与项目二融合: 将工具放入匹配的 topic 类型分类中
  │     └─ UPDATE tools SET usage_categories = [...]

Agent 需要工具时:
  ├─ 1. L1: 注入所有工具的 summary (Token 成本极低，每工具 ~30 tokens)
  ├─ 2. 当前对话类型确定后 → L2: 标签匹配筛选相关工具
  ├─ 3. 确定使用某工具后 → L3: 注入完整描述
  └─ Token 预算: L1+L2+L3 < 可配置上限 (默认 2000 tokens)
```

### 4.7 多模型调度流程（项目十二）

```
输入: 任务类型 + 所需工具 + 预算上限
  │
  ├─ 1. 过滤可用模型 (is_available=1)
  ├─ 2. 评分计算 (联合优化):
  │     总分 = w1*coding + w2*reasoning + w3*tool_compat - w4*cost
  │     权重取决于任务类型:
  │       代码任务: w1=0.5, w2=0.2, w3=0.2, w4=0.1
  │       分析任务: w1=0.1, w2=0.6, w3=0.1, w4=0.2
  ├─ 3. 成本约束: 成本 < 预算上限
  ├─ 4. 输出: 最优 (model, tools) 组合
  └─ 5. 记录选择理由到 action_memory
```

---

## 五、文件存储结构

```
data/
├── mbclaw.db                          # SQLite 主库 (WAL mode, FK ON)
│
├── memory/                            # 三层记忆文件 (Phase 2)
│   └── {project_name}/
│       ├── MEMORY.md                  # Tier 1: 持久精选事实
│       ├── YYYY-MM-DD.md             # Tier 2: 每日工作笔记
│       └── DREAMS.md                  # Tier 3: 整合日记
│
├── transcripts/                       # 会话记录 (Phase 2)
│   └── {session_id}.jsonl             # 每条消息一行 JSON (fcntl 锁)
│       └── {session_id}_part2.jsonl   # 项目一: 5MB 自动分片
│
├── snapshots/                         # 项目三: 突破备份
│   └── project_{id}/
│       └── {timestamp}/
│           ├── mbclaw.db              # 数据库热备份
│           ├── MEMORY.md              # 记忆文件快照
│           └── git_info.json          # {commit_hash, tag, message}
│
├── embeddings/                        # 项目六+十一: 向量缓存
│   └── {type}_{id}.json              # 预计算的 embedding 向量
│
└── scheduler/                         # 项目二: 调度器状态
    └── state.json                     # 上次整理时间、空闲检测状态
```

---

## 六、组件清单（需要新建的文件）

### Python 模块

```
app/
├── models/
│   ├── topic_tree.py              # 项目二: 话题树 ORM
│   ├── keyword_index.py           # 项目二: 关键词反向索引 ORM
│   ├── tool.py                    # 项目十一: 工具 ORM
│   ├── model_profile.py           # 项目十二: 模型注册 ORM
│   ├── shared_channel.py          # 项目十: 共享通道 ORM
│   ├── task_queue.py              # 项目七: 任务队列 ORM
│   └── snapshot.py                # 项目三: 快照 ORM
│
├── schemas/
│   ├── topic_tree.py
│   ├── tool.py
│   ├── model_profile.py
│   ├── shared_channel.py
│   ├── task_queue.py
│   └── snapshot.py
│
├── services/
│   ├── classify_service.py        # 项目二: 树状分类引擎
│   ├── keyword_index_service.py   # 项目二: 反向索引更新
│   ├── search_service.py          # 已有，扩展 L2(TF-IDF)+L3(embedding)
│   ├── prefetch_service.py        # 项目六: 实时记忆预调用
│   ├── snapshot_service.py        # 项目三: 快照管理
│   ├── scheduler_service.py       # 项目二: 空闲调度器
│   ├── auto_decision_service.py   # 项目四: 自动决策引擎
│   ├── collaboration_service.py   # 项目五: 双 Key 协作循环
│   ├── task_queue_service.py      # 项目七: 任务优先级队列
│   ├── shared_channel_service.py  # 项目十: 共享通道 + 反思 + 去重 + 协商
│   ├── tool_index_service.py      # 项目十一: 三层工具索引
│   ├── model_scheduler_service.py # 项目十二: 模型联合优化调度
│   ├── change_detector_service.py # 项目十三: 回滚检测
│   ├── i18n_service.py            # 项目八: 中文翻译字典
│   └── llm/
│       ├── __init__.py
│       ├── base_adapter.py        # LLM 适配器基类
│       ├── mimo_adapter.py        # 项目十三: MiMo API 适配
│       ├── openai_adapter.py      # OpenAI 兼容适配
│       └── anthropic_adapter.py   # Anthropic 适配
│
├── routers/
│   ├── topics.py                  # 项目二: 话题树 API
│   ├── semantic_search.py         # 项目六: 语义搜索 API
│   ├── snapshots.py               # 项目三: 快照 API
│   ├── auto_decision.py           # 项目四: 自动决策 API
│   ├── collaboration.py           # 项目五: 双 Key 协作 API
│   ├── tasks.py                   # 项目七: 任务队列 API
│   ├── shared_channel.py          # 项目十: 共享通道 API
│   ├── tools.py                   # 项目十一: 工具管理 API
│   ├── models.py                  # 项目十二: 模型管理 API
│   └── providers.py               # 项目十三: Provider 管理 API
│
└── middleware/
    ├── task_interrupt.py           # 项目七: 用户消息中断中间件
    └── i18n.py                     # 项目八: 中文错误消息中间件
```

### 数据文件

```
data/
├── i18n/
│   └── zh_CN.json                 # 项目八: 中文翻译字典
├── models/
│   └── default_profiles.json      # 项目十二: 预置模型能力数据
└── tools/
    └── default_tools.json         # 项目十一: 预置工具描述
```

---

## 七、关键算法伪代码

### 7.1 树状分类（项目二 — classify_service.py）

```python
def classify_session(session, topic_tree):
    # 1. 获取当前 session 的 summary + keywords
    summary = get_summary(session)
    keywords = get_keywords(session)

    # 2. 对现有话题树做匹配
    best_match = None
    best_score = 0
    for node in topic_tree.get_root_nodes():
        score = semantic_similarity(
            summary.topic + summary.conclusions,
            node.summary
        )
        if score > best_score:
            best_score = score
            best_match = node

    # 3. 判断归属
    if best_score > 0.7:  # 归入已有话题
        if node_has_failed_content(summary):
            add_failed_detail(best_match, session, summary)
        else:
            add_session_ref(best_match, session)
            update_node_summary(best_match)  # 重新生成粗略总结
    else:  # 新建话题节点
        new_node = create_topic_node(
            name=generate_topic_name(summary),
            summary=generate_brief_summary(summary, max_chars=200),
            session_refs=[session.id]
        )
        # 检测是否包含失败方案
        if node_has_failed_content(summary):
            add_failed_detail(new_node, session, summary)
```

### 7.2 突破检测（项目三 — snapshot_service.py）

```python
def check_breakthrough(project, session):
    old_dna = get_previous_dna(project)  # 从上一次快照获取
    new_dna = get_current_dna(project)

    # 检测 successful_approaches 变化
    old_success = set(json.loads(old_dna.successful_approaches))
    new_success = set(json.loads(new_dna.successful_approaches))
    added = new_success - old_success

    # 检测关键词"突破/bug fixed/解决了"
    keyword_trigger = any(
        kw in session.latest_messages.lower()
        for kw in ["突破", "bug fixed", "解决了", "终于", "搞定了"]
    )

    if added or keyword_trigger:
        create_snapshot(project, session, trigger="auto")
```

### 7.3 分层搜索（项目六 — prefetch_service.py）

```python
def prefetch_memories(project, user_message, max_context_tokens=2000):
    results = []

    # L1: 关键词反向索引 (< 10ms)
    keywords = extract_keywords_fast(user_message)
    for kw in keywords:
        index_entries = keyword_index.lookup(kw)
        for entry in index_entries:
            results.append({
                "source": "keyword_match",
                "relevance": entry.weight,
                "content": get_summary_by_session_ids(entry.session_ids)
            })

    # L2: TF-IDF 相似度 (< 100ms)
    if len(results) < 5:  # L1 不够
        tfidf_matches = tfidf_search(
            user_message,
            corpus=get_all_summaries(project),
            threshold=0.3
        )
        results.extend(tfidf_matches)

    # L3: 向量语义搜索 (< 500ms, 需要 API)
    if len(results) < 3:  # L2 还不够
        embedding = get_embedding(user_message)
        semantic_matches = vector_search(
            embedding,
            index=get_all_embeddings(project),
            top_k=5
        )
        results.extend(semantic_matches)

    # Token 预算控制
    return trim_to_token_budget(results, max_context_tokens)
```

### 7.4 模型联合优化（项目十二 — model_scheduler_service.py）

```python
def select_optimal_model(task_type, required_tools, budget):
    candidates = []

    for model in get_available_models():
        # 计算能力分
        caps = json.loads(model.capabilities)
        if task_type == "coding":
            score = 0.5*caps.get("coding",0) + 0.2*caps.get("reasoning",0)
        elif task_type == "analysis":
            score = 0.1*caps.get("coding",0) + 0.6*caps.get("reasoning",0)
        else:
            score = sum(caps.values()) / len(caps)

        # 工具兼容性
        tool_compat = json.loads(model.tool_compatibility)
        tool_score = min(tool_compat.get(t, 0.5) for t in required_tools)

        # 成本
        cost = model.cost_per_1k_input + model.cost_per_1k_output

        # 联合得分
        final_score = score * 0.6 + tool_score * 0.3 - cost * 0.1

        if cost <= budget:
            candidates.append((model, final_score))

    candidates.sort(key=lambda x: x[1], reverse=True)
    return candidates[0][0] if candidates else get_fallback_model()
```

### 7.5 回滚检测（项目十三 — change_detector_service.py）

```python
def detect_regression(project, before_commit, after_commit):
    # 获取两次 commit 之间的 diff
    diff = git_diff(before_commit, after_commit)

    # 分析每个文件的变更
    regressions = []
    for file_change in diff:
        # 检测删除/回退
        if file_change.removed_lines > file_change.added_lines * 3:
            # 大量删除 → 可能是回滚
            regressions.append({
                "file": file_change.path,
                "removed": file_change.removed_lines,
                "added": file_change.added_lines,
                "severity": "warning"
            })

        # 检测已知的 MBclaw 代码被删除
        mbclaw_files = [
            "app/services/memory_service.py",
            "app/services/transcript_service.py",
            "app/routers/memory.py",
            "app/models/action_memory.py",
        ]
        if file_change.path in mbclaw_files and file_change.removed_lines > 0:
            regressions.append({
                "file": file_change.path,
                "severity": "critical",
                "message": "MBclaw 核心文件被修改/回滚"
            })

    return regressions
```

---

## 八、依赖关系图

```
                            ┌──────────────┐
                            │  项目二       │
                            │  树状分类     │
                            │  + 反向索引   │
                            └──┬───┬───┬──┘
                               │   │   │
              ┌────────────────┘   │   └────────────────┐
              ▼                    ▼                    ▼
       ┌──────────┐        ┌──────────┐        ┌──────────┐
       │ 项目一    │        │ 项目六    │        │项目十一   │
       │ 增强日志  │        │ 实时预调用 │        │工具索引   │
       └──────────┘        └──────────┘        └──────────┘
              │                    │
              └────────┬───────────┘
                       ▼
                ┌──────────┐
                │ 项目三    │
                │ 突破备份  │
                └──────────┘

┌──────────┐     ┌──────────┐     ┌──────────┐
│ 项目七    │     │ 项目八    │     │ 项目九    │
│ 任务队列  │     │ 中文化    │     │ 删检查    │
│ (独立)    │     │ (独立)    │     │(仅Fork)  │
└─────┬────┘     └──────────┘     └──────────┘
      │
      ▼
┌──────────┐
│ 项目四    │
│ 全自动    │
└──────────┘

┌──────────┐
│ 项目十二  │
│ 模型注册  │
│ 能力评估  │
└─────┬────┘
      │
      ▼
┌──────────┐     ┌──────────┐
│ 项目五    │     │项目十三   │
│ 双Key协作 │     │MiMo集成   │
└─────┬────┘     └──────────┘
      │
      ▼
┌──────────┐
│ 项目十    │
│ 子对话协同│
└──────────┘
```

---

## 九、技术决策记录

| # | 决策 | 原因 |
|---|------|------|
| 1 | 关键词反向索引用 SQLite JSON 字段而非关联表 | 减少 JOIN，session_ids 数组不需要单独查询 |
| 2 | 向量缓存存文件不存数据库 | embedding 是大数组，SQLite 存 BLOB 效率低 |
| 3 | 空闲调度用文件标记而非数据库轮询 | 避免无谓的数据库连接 |
| 4 | LLM 适配器用抽象基类 | 方便支持 MiMo/OpenAI/Anthropic 等不同 provider |
| 5 | 话题树用 parent_id 自引用 | 天然支持树状结构，无需递归 CTE |
| 6 | 快照用 git + SQLite 双保险 | git 管代码，SQLite .backup 管数据 |
| 7 | 工具索引不用向量数据库 | MVP 用 JSON 存 embedding + 余弦相似度计算即可 |
| 8 | 子对话共享通道用 SQLite 而非消息队列 | 无需额外基础设施，单用户场景 SQLite 足够 |
| 9 | Token 预算用字符数估算而非 tokenizer | 避免引入重量级依赖，估算误差 < 10% |
| 10 | 不 fork OpenClaw | 避免维护地狱，独立演化 |
