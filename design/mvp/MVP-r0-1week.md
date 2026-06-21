# MBclaw MVP-R0（1 周版）

**版本**: r0
**日期**: 2026-06-21
**取代**: `design/mvp/MVP-v2.md`（v2 再砍 50%）

---

## 立场

> AI 写完一段对话，下次开新会话能自动看到上一段的关键事实。
> 别的都不做。

---

## 1. Core MVP 功能（5 个，硬上限）

| # | 功能 | 实现量级 | 验证方式 |
|---|---|---|---|
| F1 | 记录对话（POST 消息 → SQLite + JSONL） | service ≤80 行 | 写入后 GET 出来 |
| F2 | 总结会话（同步 LLM 出摘要） | service ≤60 行 | summaries 表 1:1 |
| F3 | 关键词提取（jieba TF-IDF top-10） | service ≤50 行 | keywords 表有数据 |
| F4 | 跨会话检索（FTS5 + 关键词命中） | service ≤60 行 | 关键词查得到上轮内容 |
| F5 | 新会话注入（top-3 摘要 + top-2 经验 ≤800 字 system message） | service ≤40 行 | 演示脚本 |

总量 ≤ 1500 行（含 schema/model/test）。

### 强制不做

| 砍 | 理由 |
|---|---|
| 向量库 / ChromaDB | 限制要求 |
| 多 Agent / Sub-Agent | 限制要求 |
| 调度器 / 队列 / 定时 | 限制要求 |
| MEMORY.md 双态 | R1+ |
| Project DNA | R1+ |
| 写入审批门 | 无 Agent 自动写 |
| 鉴权 / 多用户 | 硬编码 user_id=1 |
| i18n / Docker / K8s | 全砍 |
| 树状分类 / 失败专项表 | R1+ |
| Thinking traces / code_change | F1 只 role+content |
| 异步任务 | F2/F3 同步 |

---

## 2. 用户 3 步流程

```
Step 1: POST /sessions                           开会话（自动注入）
Step 2: POST /sessions/{id}/messages …           聊天
        POST /sessions/{id}/close                关闭（触发沉淀）
Step 3: POST /sessions                           开新会话，验证记忆
```

详细请求/响应见 §3。

---

## 3. 输入输出结构

### 数据模型（4 业务表 + experiences + 2 FTS）
见 `design/memory/MEMORY-SYSTEM-r0.md` §1。

### API 契约
见 `design/architecture/ARCH-r0.md` §4 与下方：

#### E1 `POST /sessions`
**Resp 201**
```json
{
  "session_id": 12,
  "injected_system_message": {
    "role": "system",
    "content": "【上次的关键事实】...",
    "source_sessions": [7, 5, 3]
  }
}
```

#### E2 `POST /sessions/{id}/messages`
**Body**: `{role, content}`
**Resp 201**: `{message_id, session_id, created_at}`

#### E3 `POST /sessions/{id}/close`
**Resp 200**
```json
{
  "session_id": 12,
  "summary": "...",
  "keywords": [{"keyword":"sqlite","weight":0.31}, ...],
  "experiences": [{"kind":"success","title":"...","content":"..."}],
  "stats": {"messages": 24, "summary_chars": 412, "keyword_count": 10}
}
```

#### E4 `GET /search?q=...&limit=3`
**Resp 200**
```json
{"query":"...","hits":[{"session_id":7,"summary":"...","matched_keywords":[...],"final_score":0.91}]}
```

#### E5 `GET /sessions/{id}/messages`
**Resp 200**: `[{role,content,id}, ...]`

---

## 4. 成功标准（可测试）

### 4.1 DoD
1. 5 端点 schema 正确（≥15 用例）
2. F2/F3 ≤60s
3. 端到端演示脚本通过
4. 代码 ≤1500 行
5. 核心 service 测试覆盖 ≥70%

### 4.2 端到端演示（唯一不可妥协）
见 §5 测试代码。

### 4.3 反成功标准
| 误认为进展 | 真相 |
|---|---|
| "13 项目完成 12" | 我们只有 5 |
| "覆盖率 95%" | 测的是 CRUD 不是 memory loop |
| "测试 200+" | 端到端不通过=0 |
| "支持向量/Agent" | 违反限制=FAIL |

---

## 5. 端到端演示脚本（`tests/e2e/test_memory_loop.py`）

```python
def test_mbclaw_remembers_across_sessions(client):
    # ROUND 1
    r1 = client.post("/sessions", json={"title":"选型"}).json()
    assert r1["injected_system_message"] is None
    sid1 = r1["session_id"]
    for role, content in [
        ("user", "我打算用 SQLite FTS5 做全文检索"),
        ("assistant", "FTS5 配合 jieba 分词足够 MVP"),
        ("user", "决定了，就 FTS5+jieba，不上向量库"),
    ]:
        client.post(f"/sessions/{sid1}/messages",
                    json={"role": role, "content": content})
    close1 = client.post(f"/sessions/{sid1}/close").json()
    assert close1["summary"]
    assert len(close1["keywords"]) >= 3
    assert any(k["keyword"] in ("sqlite","fts5","jieba")
               for k in close1["keywords"])

    # ROUND 2 — 关键验证
    r2 = client.post("/sessions", json={"title":"继续"}).json()
    sid2 = r2["session_id"]
    inj = r2["injected_system_message"]
    assert inj is not None
    assert f"#{sid1}" in inj["content"]
    assert any(kw in inj["content"].lower()
               for kw in ["sqlite","fts5","jieba"])
    assert len(inj["content"]) <= 800

    # ROUND 3 — 注入在历史可见
    msgs = client.get(f"/sessions/{sid2}/messages").json()
    assert msgs[0]["role"] == "system"
    assert msgs[0]["content"] == inj["content"]
```

通过=MVP 成立；不通过=失败。
