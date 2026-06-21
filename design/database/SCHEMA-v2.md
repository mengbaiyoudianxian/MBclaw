# 数据库 Schema v2 — 8 张表，收敛版

**版本**: v2
**日期**: 2026-06-21
**目标**: 把 24 张表收敛到 8 张

---

## 1. 表清单

| 表 | 用途 | 与现 Lite 关系 |
|---|---|---|
| `users` | 用户（MVP 单用户硬编码 id=1） | 保留 |
| `projects` | 项目（一个用户 N 个项目） | 保留 |
| `sessions` | 会话（一个项目 N 个会话） | 保留 |
| `messages` | 消息（一个会话 N 条消息） | 保留 |
| `summaries` | 会话结束摘要（1:1） | 保留 |
| `keywords` | 关键词反向索引 | 保留 |
| `project_dna` | goals / successful / failed | 保留 |
| `memory_audit` | 写入操作审计（替代 ApprovalLog/PendingApproval） | 新增（合并简化） |

---

## 2. DDL 草案

```sql
CREATE TABLE users (
  id INTEGER PRIMARY KEY,
  name TEXT UNIQUE NOT NULL,
  created_at TEXT NOT NULL
);

CREATE TABLE projects (
  id INTEGER PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id),
  name TEXT NOT NULL,
  description TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  UNIQUE(user_id, name)
);

CREATE TABLE sessions (
  id INTEGER PRIMARY KEY,
  project_id INTEGER NOT NULL REFERENCES projects(id),
  session_number INTEGER NOT NULL,
  title TEXT,
  status TEXT NOT NULL CHECK(status IN ('active','completed')),
  started_at TEXT NOT NULL,
  ended_at TEXT
);

CREATE TABLE messages (
  id INTEGER PRIMARY KEY,
  session_id INTEGER NOT NULL REFERENCES sessions(id),
  role TEXT NOT NULL CHECK(role IN ('user','assistant','system','tool')),
  content TEXT NOT NULL,
  thinking TEXT,                  -- 可空：AI thinking traces
  tool_calls_json TEXT,           -- 可空：工具调用结构
  created_at TEXT NOT NULL
);

CREATE TABLE summaries (
  id INTEGER PRIMARY KEY,
  session_id INTEGER UNIQUE NOT NULL REFERENCES sessions(id),
  summary TEXT NOT NULL,
  created_at TEXT NOT NULL
);

CREATE TABLE keywords (
  id INTEGER PRIMARY KEY,
  session_id INTEGER NOT NULL REFERENCES sessions(id),
  keyword TEXT NOT NULL,
  weight REAL NOT NULL,
  created_at TEXT NOT NULL
);
CREATE INDEX idx_keywords_kw ON keywords(keyword);

CREATE TABLE project_dna (
  id INTEGER PRIMARY KEY,
  project_id INTEGER UNIQUE NOT NULL REFERENCES projects(id),
  goals_json TEXT NOT NULL DEFAULT '[]',
  successful_json TEXT NOT NULL DEFAULT '[]',
  failed_json TEXT NOT NULL DEFAULT '[]',
  updated_at TEXT NOT NULL
);

CREATE TABLE memory_audit (
  id INTEGER PRIMARY KEY,
  op TEXT NOT NULL CHECK(op IN ('add','replace','remove')),
  target TEXT NOT NULL,              -- e.g. 'MEMORY.md' | 'USER.md'
  payload TEXT NOT NULL,             -- 写入内容快照
  approved INTEGER NOT NULL,         -- 0=pending, 1=auto, 2=user
  origin TEXT NOT NULL,              -- 'agent' | 'user' | 'system'
  created_at TEXT NOT NULL
);

-- FTS5 虚拟表（M4 检索基础）
CREATE VIRTUAL TABLE messages_fts USING fts5(
  content,
  content='messages',
  content_rowid='id',
  tokenize='unicode61'
);
-- 触发器自动同步
CREATE TRIGGER messages_ai AFTER INSERT ON messages BEGIN
  INSERT INTO messages_fts(rowid, content) VALUES (new.id, new.content);
END;
CREATE TRIGGER messages_ad AFTER DELETE ON messages BEGIN
  INSERT INTO messages_fts(messages_fts, rowid, content) VALUES('delete', old.id, old.content);
END;
CREATE TRIGGER messages_au AFTER UPDATE ON messages BEGIN
  INSERT INTO messages_fts(messages_fts, rowid, content) VALUES('delete', old.id, old.content);
  INSERT INTO messages_fts(rowid, content) VALUES (new.id, new.content);
END;
```

---

## 3. 被移除/延期的表

| 旧表 | 处置 | 说明 |
|---|---|---|
| ActionMemory | 延期 R2 | 当前无 Agent 真实使用 |
| Tool / ModelProfile | 延期 R2 | 工具索引、模型调度未到时机 |
| ClassificationNode | 延期 R2 | 树状分类无数据支撑 |
| Snapshot | 简化 | 改为文件级 SQLite backup，不入表 |
| SkillCard | 延期 R2 | 等 Agent 真实运作再说 |
| PendingApproval / ApprovalLog | 合并 | → `memory_audit` 单表 |
| ExternalIntegration | 延期 R3 | 11 平台延后 |
| Feedback / UserProfile / PositivePattern | 延期 R2 | 心理画像不入 Core |
| ThoughtCollision | 永久移出 | 见 MVP-v2 §1.X1 |
| Utopia* (4 张) | 永久移出 | 见 MVP-v2 §1.X2 |
| TaskQueue | 延期 R2 | 当前无并发 |

**总数**：24 → 8（含 1 张 FTS5 虚拟表 = 8 业务表 + 1 索引表）。

---

## 4. 索引与查询模式

- 检索主路径：`messages_fts MATCH ?` JOIN `messages` JOIN `sessions`。
- 关键词跨会话：`keywords WHERE keyword IN (?)` GROUP BY `session_id`。
- DNA 单点：`project_dna WHERE project_id = ?`（永远 1 行）。

---

## 5. 迁移策略（从现 Lite）

- R1 直接新建 schema（不做兼容迁移）——MVP 数据无生产价值。
- 提供一次性 `scripts/export_legacy.py` 把旧数据导出 JSON 备份，**导入 Memory 仓库归档**，不进 Core。
