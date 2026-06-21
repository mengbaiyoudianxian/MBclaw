# MBclaw R0 详尽开发计划（弱 AI 可执行版）

**版本**: r0
**日期**: 2026-06-21
**适用执行方**: OpenHands / 任意 AI 编码代理
**前置阅读**: ARCH-r0 / MVP-r0-1week / MEMORY-SYSTEM-r0 / AGENT-r0

**设计目标**：执行 AI 不需要思考，只需照做。每个任务有：输入 / 步骤 / 验证 / 输出 / 不允许做什么。

---

## 0. 顶层规则（钉死在 PR 模板）

1. **1 PR 1 Task**，commit 标题前缀 `[T*.*]`
2. **不准加 requirements** —— 新依赖必须先在 Design 仓 issue
3. **没有对应单测的 PR 直接拒**
4. **超行数预算 → block**
5. **业务码直 import 模型表 = reject**（必须走 MemoryRepo）
6. **改 e2e 断言阈值 = 承认 MVP 失败**

---

## 1. Core 任务树（编号即执行顺序）

### Phase 0 — 仓库与骨架（2h）

#### T0.1 创建 r0 分支
- 仓库: MBclaw-Lite
- 命令: `git checkout main && git pull && git checkout -b r0`
- 验证: `git branch` 显示 `* r0`
- 不允许: 在 r0 直接 rm 旧文件（保留 main 参考）
- 依赖: 无

#### T0.2 清空 app/
- `git rm -rf app/ && git rm -rf tests/`
- `mkdir -p app tests/unit tests/e2e data`
- `touch app/__init__.py tests/__init__.py`
- 验证: `ls app/` 只有 `__init__.py`
- 不允许: 保留任何旧 service / router / model

#### T0.3 写 requirements.txt
```
fastapi==0.115.0
uvicorn[standard]==0.32.0
sqlalchemy==2.0.35
pydantic==2.9.2
jieba==0.42.1
httpx==0.27.2
pytest==8.3.3
pytest-cov==5.0.0
```
- 不允许: chromadb / celery / redis / langchain / llama-index

#### T0.4 写 .env.example
```
MBCLAW_LLM_BASE_URL=https://api.openai.com/v1
MBCLAW_LLM_API_KEY=sk-xxx
MBCLAW_LLM_MODEL=gpt-4o-mini
MBCLAW_DB_PATH=data/mbclaw.db
MBCLAW_AUTO_APPROVE_WRITE=true
```
- 不允许: MIMO 专用变量

---

### Phase 1 — 数据层（6h）

#### T1.1 app/db.py（≤80 行）
- DATABASE_URL 从 env 读
- `engine = create_engine(url, connect_args={"check_same_thread": False})`
- 启动 PRAGMA: `journal_mode=WAL` / `synchronous=NORMAL` / `cache_size=-20000` / `temp_store=MEMORY`
- `SessionLocal` / `Base` / `get_db()` / `init_db()`
- `init_db()` 末尾 `executescript(open('app/schema/fts.sql').read())`
- 验证: `python -c "from app.db import init_db; init_db()"` 生成 db 文件
- 不允许: alembic / 业务查询

#### T1.2 app/models.py（≤120 行）
5 个 model:
- `Session(id, title, status, started_at, ended_at)`
- `Message(id, session_id FK, role, content, created_at)`
- `Summary(id, session_id FK UNIQUE, summary, created_at)`
- `Keyword(id, session_id FK, keyword, weight)`
- `Experience(id, session_id FK, kind, title, content, keywords_json, created_at, last_recalled_at NULL, recall_count default 0)`
- 验证: `sqlite3 data/mbclaw.db ".tables"` 显示 5 张
- 不允许: 给 model 挂业务方法；加 dna/project/user/approval 表

#### T1.3 app/schema/fts.sql
- 2 个 FTS5 虚表（messages_fts, experiences_fts）
- 6 个触发器（每表 INSERT/DELETE/UPDATE）
- 验证: `SELECT name FROM sqlite_master WHERE type='trigger'` 应有 6 个
- 不允许: 多于 2 个 FTS 表

---

### Phase 2 — LLM 客户端（3h）

#### T2.1 app/llm.py（≤120 行）
```python
class LLMClient:
    def __init__(self, base_url, api_key, model): ...
    def summarize_session(self, messages: list[dict]) -> LLMOutput:
        # POST /chat/completions, response_format={"type":"json_object"}
        # 失败重试 1 次，仍失败抛 LLMError
```
Prompt 模板（写死）：
```
分析以下对话，严格输出 JSON：
{
  "summary": "≤300字概括用户目标/达成结论/未决问题",
  "keywords": ["最多10个"],
  "experiences": [{"kind":"success|failure|lesson","title":"≤80字","content":"≤500字"}]
}
experiences 最多 5 条。没有则空数组。
对话：
{messages_text}
```
pydantic 校验:
```python
class LLMOutput(BaseModel):
    summary: str = Field(max_length=400)
    keywords: list[str] = Field(max_length=10)
    experiences: list[Experience] = Field(max_length=5)
```
- 验证: mock httpx 返回合法 JSON
- 不允许: MiMo 特殊路径；调业务模块；多 prompt 模板

---

### Phase 3 — 记忆抽象层（8h，最关键）

#### T3.1 MemoryRepo.write_session_memory
```python
class MemoryRepo:
    def __init__(self, db_session): self.db = db_session
    def write_session_memory(self, sid, summary, keywords, experiences) -> None:
        # 原子事务: summary + keywords + experiences[≤5]
        # 提交后调 _maybe_evict_experiences()
```
- 不允许: 调 LLM；接 HTTP；超 5 条 experiences

#### T3.2 MemoryRepo.query（双路召回 + 打分）
```python
def query(self, q: str, top_n: int = 3) -> list[MemoryHit]:
    # A. messages_fts MATCH + JOIN summaries → max(fts_score) per session
    # B. keywords IN (jieba.cut_for_search(q)) → 命中数
    # final = 0.6 * fts_norm + 0.4 * kw_score
    # return top_n
```
- 不允许: 调 LLM；上向量

#### T3.3 query_experiences + render_injection
```python
def query_experiences(self, q, top_n=2) -> list[Experience]:
    # experiences_fts MATCH q
    # score = 0.7 * fts_norm + 0.3 * (log(recall_count+1) + kind_priority)
    # kind_priority = {failure:1.0, lesson:0.8, success:0.5}
    # 副作用: UPDATE last_recalled_at=now(), recall_count+=1

def render_injection_for_new_session(self, exclude_sid) -> str | None:
    # 1. self-prime: 最近 1 closed session 的 summary+top5 keywords 作 query
    # 2. summaries_hits = query(query, 3)
    # 3. exp_hits = query_experiences(query, 2)
    # 4. 拆 failure/lesson vs success
    # 5. 模板渲染，≤800 字符硬截断
    # 6. 空区块省略整段
    # 7. 全空 → None
```
模板：
```
【上次的关键事实】
- [#{sid}] {summary[:120]}  关键词: {kw1, kw2, kw3}

【避免重复的失败】
- ⚠️ [#{sid}] {title}
- 💡 [#{sid}] {title}

【已验证的成功】
- ✅ [#{sid}] {title}
```
- 不允许: 调 LLM；超 3 区块；超 800

#### T3.4 _maybe_evict_experiences
- 仅当 `count > 1000` 时执行
- 归档到 `data/archive/experiences-YYYY-MM.jsonl` 再删
- 不允许: 后台定时；自动跑（必须由 write_session_memory 调用）

---

### Phase 4 — 沉淀管线（3h）

#### T4.1 app/pipeline.py（≤80 行）
```python
def close_session(db, sid, llm) -> dict:
    # 1. 加载 messages
    # 2. 已 closed → 幂等返回
    # 3. llm_out = llm.summarize_session(messages)
    # 4. jieba TF-IDF top-10（与 llm_out.keywords 合并去重，llm 权重 1.0 / jieba 0.5）
    # 5. MemoryRepo(db).write_session_memory(...)
    # 6. session.status='closed'; ended_at=now()
    # 7. return {summary, keywords, experiences, stats}
```
- 不允许: 异步；调方法外 service

---

### Phase 5 — API 层（4h）

#### T5.1 app/api.py（≤300 行）
5 端点（路径 / 行为 / 响应详见 MVP-r0 §3）：
- `POST /sessions` → 建 + 注入
- `POST /sessions/{sid}/messages` → 写 + JSONL
- `POST /sessions/{sid}/close` → pipeline
- `GET /sessions/{sid}/messages`
- `GET /search?q=&limit=`
- 不允许: 加端点；管理后台；鉴权

#### T5.2 app/main.py（≤80 行）
- `app = FastAPI(...)` + lifespan: init_db
- DI: `get_db` / `get_llm`
- `GET /health`
- JSONL helper: `append_transcript(sid, msg)` 用 fcntl.LOCK_EX
- 不允许: 多 middleware（仅 CORS）

---

### Phase 6 — 测试（6h，与 Phase 1-5 并行）

#### T6.1 单测 ≥40 用例
- test_db (4) / test_models (5) / test_llm (6) / test_memory_write (6) / test_memory_query (6) / test_memory_render (6) / test_memory_evict (3) / test_pipeline (4)
- 强制 mock LLM

#### T6.2 tests/e2e/test_memory_loop.py
（脚本见 MVP-r0-1week §5）
关键断言：
- ROUND 1 首次 inj is None
- ROUND 1 close 返回 summary + keywords ≥ 3
- ROUND 2 inj 含 #sid1 + 至少 1 个 (sqlite|fts5|jieba)
- ROUND 2 inj 长度 ≤ 800
- ROUND 3 GET 首条 role=system
- 不允许: 改阈值；跳 assert

---

### Phase 7 — 验收（2h）

#### T7.1 DoD 5 条
1. 5 端点 schema → curl 实测
2. F2/F3 ≤60s → `time pytest tests/e2e`
3. e2e 通过
4. `find app -name '*.py' | xargs wc -l | tail -1` ≤ 1500
5. `pytest --cov=app/memory --cov=app/pipeline --cov=app/llm --cov-fail-under=70`

#### T7.2 打 tag
```
git tag v0.0.1-mvp
git push --tags
```

---

## 2. Design 任务

| ID | 内容 | 触发 |
|---|---|---|
| D1 | 文档与代码漂移检查（每 PR review） | 持续 |
| D2 | R1 触发信号埋点与监控（命中率 / 解析失败率 / sessions 数） | R1 上线后 |
| D3 | ReflectionAgent 设计稿（仅设计，不实施） | 用户主动提"代我做"场景 |
| D4 | 竞品差异化论证 vs mem0/Letta/Zep/Cognee | R1 演示前 |

---

## 3. Memory 任务

| ID | 内容 | 时机 |
|---|---|---|
| M1 | 物理迁出 14 个旧 services 到 `MBclaw-Memory/drafts/legacy/<分类>/` | T0.2 之前一次性 |
| M2 | 一次性导出 main 分支 SQLite 数据到 Memory | M1 之后 |
| M3 | 实施中新出现的失败 / 否决随时归档 | 持续 |

---

## 4. 优先级排序

```
P0  T0.1 → T0.2 → T0.3 → T0.4              (2h)
P0  T1.1 → T1.2 → T1.3                      (6h)
P0  T2.1                                    (3h)
P0  T3.1 → T3.2 → T3.3 → T3.4               (8h)
P0  T4.1                                    (3h)
P0  T5.1 → T5.2                             (4h)
P0  T6.1（与各 Phase 并行）                  (6h)
P0  T6.2                                    (2h)
P0  T7.1 → T7.2                             (2h)
─────
P1  M1（与 T0.2 同步由 OpenHands 跑）        (2h)
P1  M2                                      (1h)
P2  D1 / D3 持续
P3  D4（R1 ship 前）
```

---

## 5. 依赖关系（图）

```
T0.1 → T0.2 → T0.3 → T0.4
                │
                ▼
              T1.1 → T1.2 → T1.3
                            │
              ┌─────────────┤
              ▼             ▼
            T2.1          T3.1 → T3.2 → T3.3
                            │              │
                            └→ T3.4        │
                                │          │
                                └────┬─────┘
                                     ▼
                                   T4.1
                                     │
                                     ▼
                                   T5.1 → T5.2
                                     │
                                     ▼
                          T6.1（沿途） + T6.2
                                     │
                                     ▼
                                   T7.1 → T7.2

并行: M1 与 T0.2 同时
```

---

## 6. 第一周计划

| 天 | 任务 | 完成判定 |
|---|---|---|
| D1 周一 | T0.1-T0.4 + T1.1-T1.3 + M1 + M2 | `init_db()` 成功；旧码归档 |
| D2 周二 | T2.1 + test_db/test_models/test_llm | pytest tests/unit -q 通过 |
| D3 周三 | T3.1 + T3.2 + 单测 | MemoryRepo 写入与召回单测全绿 |
| D4 周四 | T3.3 + T3.4 + 单测 | render_injection 含 ≤800 断言通过 |
| D5 周五 | T4.1 + T5.1 + T5.2 + 路由集成测 | uvicorn 启动，5 端点 200 |
| D6 周六 | T6.2 端到端 + bugfix | tests/e2e 通过 |
| D7 周日 | T7.1 验收 + T7.2 tag + 文档同步 | tag v0.0.1-mvp 推送 |

**缓冲规则**: D6 通不过 → 砍 D7 文档同步，专心修 bug。**绝不**砍测试或加新功能。

---

## 7. 完整路线图

```
R0  本周         冻结+归档+干净分支                              [当前]
R1  +1 周        MVP 5 端点 e2e 通过 → v0.0.1-mvp
R1.1 +2 周       埋点上线 + 修文档
R1.2 +4 周       OpenAPI + Docker
R2  信号驱动     按触发阈值，绝不批量上：
                 ├ C5 命中率 < 60% → MEMORY.md 双态
                 ├ C4 召回率 < 70% → 向量增强
                 ├ 多项目混淆 → projects + DNA
                 ├ 摘要质量 < 75% → Reflection 再加工
                 ├ sessions > 500 → 树状分类
                 ├ ≥3 provider → 多模型调度
                 ├ LLM 解析失败 > 10% → prompt 优化
                 └ 用户"代我做"场景 → ReflectionAgent（单步）
R3  信号驱动     ├ 单平台 DAU > 50 → 该平台网关
                 ├ 海外用户 ≥ 10 → i18n
                 ├ Lite 稳 3 月 → miclaw/Android
                 └ 单机扛不住 → K8s
```

**铁律**: R2/R3 每项独立 PR、独立信号、独立论证、独立 ship。**禁止"R2 一次上 7 个"**。

---

## 8. 风险清单（概率 × 影响排序）

### 🔴 R1：弱 AI 偷加功能 / 引 LangChain
- 概率: 高  影响: 致命
- 缓解: CI grep `langchain|chromadb|celery|redis|qdrant` 拦截；CTO PR review

### 🔴 R2：弱 AI 合并多 Phase 提交（巨型 PR）
- 概率: 中高  影响: 无法 review
- 缓解: PR diff > 400 行直接拒；commit 前缀 `[T*.*]` 强制

### 🔴 R3：T3.3 注入超 800 字符
- 概率: 高  影响: e2e 失败
- 缓解: 单测含极长输入；截断策略 = 按区块整体丢弃

### 🟡 R4：LLM 返回非法 JSON
- 概率: 高  影响: pipeline 半成品
- 缓解: `response_format={"type":"json_object"}`；重试 1 次；失败仅写 summary 占位；错误率 > 10% 触发 R2

### 🟡 R5：jieba 在罕见词上劣化
- 概率: 中  影响: 召回率下降
- 缓解: 测试含中英混合；召回 < 70% 触发 R2 向量

### 🟡 R6：弱 AI import 模型表绕 MemoryRepo
- 概率: 中  影响: 风险 #2 复现
- 缓解: CI grep `from app.models import (Summary|Keyword|Experience)` 在 api/pipeline 中 → reject

### 🟡 R7：测试调真 LLM 烧钱 / CI 不稳
- 概率: 高  影响: 月底账单
- 缓解: tests/unit 强制 mock；tests/e2e 默认 mock，加 `@pytest.mark.live_llm` 本地手工

### 🟢 R8：SQLite WAL 在某 FS 失效
- 概率: 低  影响: 写并发慢
- 缓解: 启动检查 print `journal_mode`

### 🟢 R9：模板 emoji 失控
- 概率: 中  影响: 视觉污染
- 缓解: 模板写死，PR 时对比

### 🟢 R10：用户中途想加新功能
- 概率: 极高  影响: 又膨胀
- 缓解: 新需求一律进 Design；R0/R1 内答复"R1 ship 后再说"

---

## 9. 给执行 AI 的 PR 模板（必须填写）

```markdown
## Task ID
[T*.*]

## What changed (≤3 句)


## Affected files (列出每个文件 + 行数变化)


## Tests added/modified


## DoD checklist
- [ ] commit 标题前缀 [T*.*]
- [ ] 未引入新依赖（或已在 Design 仓批准）
- [ ] 单测覆盖本任务
- [ ] 文件行数符合约束
- [ ] 未在 api/pipeline 直 import 模型表
- [ ] e2e 断言阈值未改
```
