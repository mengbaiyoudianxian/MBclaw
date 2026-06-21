# 长期记忆系统设计 — R0

**版本**: r0
**日期**: 2026-06-21
**取代**: 无（首版）
**配套**: `design/architecture/ARCH-r0.md` / `design/mvp/MVP-r0-1week.md`

---

## 0. 一句话

**关会话时同步出 1 次 LLM，产 summary + keywords + 0~5 条 experiences；开新会话同步召回 top-3 摘要 + top-2 失败/教训，拼成 ≤800 字 system message 注入。**

无后台任务，无向量库，无多层索引。

---

## 1. SQLite 表（5 业务表 + 2 FTS）

```sql
sessions      (id, title, status, started_at, ended_at)
messages      (id, session_id, role, content, created_at)
summaries     (id, session_id UNIQUE, summary, created_at)
keywords      (id, session_id, keyword, weight)

-- 新增：唯一承载经验沉淀 + 失败记录的表
CREATE TABLE experiences (
  id INTEGER PRIMARY KEY,
  session_id INTEGER NOT NULL REFERENCES sessions(id),
  kind TEXT NOT NULL CHECK(kind IN ('success','failure','lesson')),
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  keywords_json TEXT NOT NULL DEFAULT '[]',
  created_at TEXT NOT NULL,
  last_recalled_at TEXT,
  recall_count INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX idx_exp_kind ON experiences(kind);
CREATE INDEX idx_exp_recall ON experiences(last_recalled_at);

CREATE VIRTUAL TABLE messages_fts USING fts5(
  content, content='messages', content_rowid='id', tokenize='unicode61'
);
CREATE VIRTUAL TABLE experiences_fts USING fts5(
  title, content, content='experiences', content_rowid='id', tokenize='unicode61'
);
-- 各自 3 个 INSERT/UPDATE/DELETE 触发器同步
```

**为什么不分 success/failure/lesson 三表**：增长控制、召回路径、迁移成本都 ×3，`kind` 字段过滤足矣。

---

## 2. 写入流程（同步，无队列）

```
POST /sessions/{id}/close
  → pipeline.close(sid):
      1. load messages
      2. llm.summarize_session(messages) → JSON
         {summary, keywords, experiences[≤5]}
      3. memory.write_session_memory(sid, ...)
         - summaries: 1 row
         - keywords:  N rows
         - experiences: 0~5 rows
      4. sessions.status = 'closed'
      5. (顺手) 经验淘汰检查（见 §5）
      6. return {summary, keywords, experiences, stats}
```

### 约束
- 经验 ≤5 条/session（防 LLM 膨胀输出）
- `experiences.content` 截断到 500 字
- LLM 返回非法 JSON → 仅写 summary+keywords，experiences 跳过，记 warning

### LLM Prompt（写死，唯一一处 LLM 调用）
```
分析以下对话，输出 JSON：
{
  "summary": "≤300字概括用户目标/达成结论/未决问题",
  "keywords": ["最多10个"],
  "experiences": [
    {"kind":"success|failure|lesson","title":"≤80字","content":"≤500字"}
  ]   // 最多 5 条；没有则空数组
}
```

---

## 3. 读取流程（开新会话注入）

```
POST /sessions
  → memory.render_injection_for_new_session():
      1. 取最近 1 个 closed session 的 (summary + top-5 keywords) 作 query
      2. 召回：
         A. summaries_hits = fts5(messages.content) + keywords 命中 → top-3
         B. experiences_hits = fts5(experiences) → top-2，优先 failure/lesson
      3. UPDATE experiences SET last_recalled_at=now(), recall_count+=1
         WHERE id IN (...)
      4. render(A, B) → system message ≤800 字符
      5. 写入 messages 表 (role='system')
```

### 注入模板（写死）
```
【上次的关键事实】
- [#7] 决定用 SQLite FTS5 + jieba（kw: sqlite, fts5, jieba）
- [#5] ...

【避免重复的失败】
- ⚠️ [#6] 试图用 ChromaDB 但部署复杂
- 💡 [#4] 教训：分层搜索没有数据支撑

【已验证的成功】
- ✅ [#3] 关键词 + FTS5 召回率 78%
```

3 区块独立；无召回则省略整段。总长硬截断 800 字符。

### 打分（写死）
```
summary_score = 0.6 * normalize(fts_score) + 0.4 * keyword_hit_ratio
experience_score = 0.7 * normalize(fts_score) + 0.3 * (recall_count_log + kind_priority)
  kind_priority = {failure: 1.0, lesson: 0.8, success: 0.5}
```

---

## 4. 经验总结机制（轻量）

**关会话时 LLM 一次调用同时产 summary + keywords + experiences**。

不做：
- ❌ 定时后台总结（无调度器）
- ❌ 跨会话再总结
- ❌ Curator 自动归档
- ❌ SkillCard
- ❌ Dreaming

**质量保障**：
- pydantic JSON schema 校验
- 单元测试用 mock LLM 验证 prompt 稳定输出
- 异常 → 仅 summary 落库，记 warning

---

## 5. 数据增长控制（朴素淘汰）

| 数据 | 策略 |
|---|---|
| `messages` / transcripts | **永不删** |
| `summaries` / `keywords` | **永不删**（极小） |
| **`experiences`** | 超过 1000 条触发软淘汰 |

```sql
DELETE FROM experiences
WHERE id IN (
  SELECT id FROM experiences
  WHERE (last_recalled_at IS NULL AND created_at < datetime('now','-90 days'))
     OR (last_recalled_at < datetime('now','-90 days') AND recall_count = 0)
  ORDER BY created_at ASC
  LIMIT 100
);
```

被删 → 追加 `data/archive/experiences-{YYYY-MM}.jsonl`，不丢失。
触发时机：`pipeline.close()` 写完 experiences 后**同事务**检查。**无后台任务**。

---

## 6. 查询优化（最少必要）

| 路径 | 优化 |
|---|---|
| messages 写入 | FTS5 触发器 |
| summaries 召回 | messages_fts MATCH + PK JOIN |
| experiences 召回 | experiences_fts MATCH + idx_exp_kind |
| 关键词命中 | idx_kw_keyword |
| 淘汰扫描 | idx_exp_recall |

### SQLite PRAGMA（启动时一次）
```sql
PRAGMA journal_mode=WAL;
PRAGMA synchronous=NORMAL;
PRAGMA cache_size=-20000;  -- 20MB
PRAGMA temp_store=MEMORY;
```

不做：物化视图、多列复合索引、计划缓存、连接池调优。

---

## Design（可选优化，触发信号）

| 项 | 信号 |
|---|---|
| 跨 session 经验合并 | 召回出现重复失败 |
| MEMORY.md 双态架构 | 命中率 < 60% |
| Project DNA + projects 表 | 多项目并存混淆 |
| Reflection 再加工 | experiences 质量 < 75% |
| 向量召回 | FTS5 召回 < 70%（实测） |
| BM25 rerank | top-3 经常含无关 |
| 树状分类 | sessions > 500 |

---

## Memory（废弃，禁止重启除非新证据）

- 多层索引 L1/L2/L3（见 `MBclaw-Memory/experiments/failed/premature-layered-search`）
- Curator 30/90 天自动归档
- SkillCard + SHA256 去重
- failed_approaches 独立表
- PendingApproval 多维评分写入门
- Dreaming 后台整合
- ChromaDB 向量库（R0 期违反限制）
- 多种记忆形态并存（碎片化反模式）
