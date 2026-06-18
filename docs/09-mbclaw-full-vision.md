# 09 — MBclaw 完整愿景：13 项目改造方案

> 基于 MBclaw 现有架构 + OpenClaw 参考 + 用户完整想法
> 每项包含：需求分析、对比 OpenClaw 做法、MBclaw 差距、修改方案、潜在问题

---

## 项目一：详细日志备份

### 需求
完整记录每次对话的：改了什么代码、用户与 AI 对话内容、AI 思考了什么（thinking traces），全部详细备份到独立文件。每个文件大小合适，方便后续整理分类。

### OpenClaw 做法
- JSONL transcript：`<sessionId>.jsonl`，每行一条消息（role + content + timestamp）
- 文件级 fcntl 写锁
- Session 完整生命周期日志

### MBclaw 当前状态
- ✅ `data/transcripts/{session_id}.jsonl` 已记录 role + content + timestamp
- ❌ 不记录"改了什么代码"（code diff/changes）
- ❌ 不记录"AI 思考了什么"（thinking traces，OpenHands 的 `<thinking>` 块）
- ❌ 文件大小不可控，没有自动分割

### 需要修改
1. **记录代码变更**：在 transcript 中增加 `type: "code_change"` 记录，包含 file_path、diff、before/after
2. **记录 thinking**：在每条 assistant 消息中增加 `thinking` 字段，存储 AI 推理过程
3. **文件分割**：每个文件最大 5MB，超出自动创建 `{session_id}_part2.jsonl`
4. **schema 升级**：
```json
{
  "type": "message | code_change | thinking | summary | decision",
  "role": "user | assistant | system",
  "content": "...",
  "thinking": "...",
  "code_change": {"file": "...", "diff": "..."},
  "timestamp": "..."
}
```

### 问题/风险
- thinking traces 可能很大（OpenHands 的 thinking 块可达数 KB）
- 代码 diff 存储会导致文件膨胀很快，需要压缩或只存关键变更

---

## 项目二：空闲时自动整理对话

### 需求
空闲时刻调用 API 自动分析所有对话，按主题类型分类，形成树状结构（粗略总结 → 详细内容），包含行不通方案的细节总结。给每个对话提取关键词，AI 需要时可根据关键词搜索唤醒详细记忆。

### OpenClaw 做法
- **Dreaming**：后台整合通道，score → filter → promote
- **Memory Wiki**：将知识编译成结构化页面（claims + evidence + contradiction tracking + freshness）
- **Memory Flush**：上下文压缩前静默保存
- **Daily notes**：每天的工作笔记自动加载

### MBclaw 当前状态
- ✅ Dreaming（`POST /api/projects/{id}/memory/dream`）：整合最近 7 天总结 → MEMORY.md
- ✅ Memory Flush（session complete 时自动执行）
- ✅ 关键词提取（jieba + TF-IDF）
- ❌ 不做树状分类（粗略→详细的分支结构）
- ❌ 不做"行不通方案"的专项标记和总结
- ❌ 不自带定时/空闲触发（需要外部 cron 调用 /memory/dream）

### 需要修改
1. **树状分类引擎**：按主题聚类对话，生成层级结构
```
Project/
  ├── Topic: 数据库设计/
  │   ├── Summary: 选择了 SQLite WAL 模式
  │   ├── Session #3: 详细讨论了 PostgreSQL vs SQLite
  │   └── Failed: 尝试了 MySQL → 放弃（原因：太重）
  └── Topic: API 设计/
      ├── Summary: REST 风格，FastAPI
      └── Session #5: 讨论了 GraphQL vs REST
```
2. **失败方案专项存储**：在 Project DNA 中增加 `failed_approaches_detail` 字段（结构化 JSON，不只是文本）
3. **空闲调度器**：`app/services/scheduler.py`，检测 API 空闲时自动触发整理
4. **关键词反向索引**：`keyword → [session_ids]` 映射表，O(1) 查找

### 问题/风险
- "空闲"的定义：多久没请求算空闲？需要可配置
- 树状分类需要 LLM 调用（本地 jieba 做不到语义分类），成本需考虑
- 中文分类准确度依赖 LLM 质量

---

## 项目三：突破时备份

### 需求
项目有突破性进展时，自动做完整快照备份，防止后期修改破坏成果。

### OpenClaw 做法
- Session 自动归档（completed → JSONL 只读）
- 没有 project-level 的自动 git 快照

### MBclaw 当前状态
- ❌ 没有项目快照/版本管理
- ✅ SQLite 数据库（可以用 `VACUUM INTO` 热备份）

### 需要修改
1. **Git 自动快照**：session complete 时检测 DNA 变更，如果 `successful_approaches` 增加 → 自动 `git commit + tag`
2. **数据库热备份**：`sqlite3 mbclaw.db ".backup mbclaw_2026-06-18.db"`
3. **触发条件**：DNA 的 successful_approaches 变化 / 用户标记 / 关键词匹配"突破/bug fixed/解决了"
4. **存储**：`data/snapshots/project_{id}/{timestamp}/`

### 问题/风险
- 以谁的身份 git commit？（需要 git config）
- 自动判断"突破"不准确，可能会产生大量无用备份

---

## 项目四：全自动模式

### 需求
用户要求全自动时，不需要用户确认就自己判断哪个方案更好，自动执行。用户没反应就自动研究其他选项，尽可能做几个成品让用户选。

### OpenClaw 做法
- **Commitments**：推断的短期后续行动（"check in after the interview"）
- **DMs pairing**：未知发送者需配对审批
- 没有"全自动无障碍模式"

### MBclaw 当前状态
- ❌ 没有自动决策模式
- ❌ 没有多方案并行执行
- ❌ 所有行动需要用户显式确认

### 需要修改
1. **自动决策引擎**：`app/services/auto_decision.py`
   - 遇到选择时：LLM 自评各方案优劣 → 选最优 → 执行
   - 记录决策理由到 action_memory
2. **多方案并行**：多个 sub-agent 同时跑不同方案
3. **自动研究**：用户不回复时，agent 自己搜索/测试各选项
4. **成品展示**：做完后整理对比，等待用户选择
5. **安全边界**：危险操作（rm -rf /、修改系统配置）必须确认

### 问题/风险
- 安全：自动执行可能有破坏性（需要与项目九的检查删除配合）
- 成本：多方案并行 = N 倍 API 调用
- 判断标准不明确：什么算"更好的方案"

---

## 项目五：双 Key 协作

### 需求
Key1 做产品，Key2 分析选择/评价/找 bug/给改进方案，Key1 继续改。循环 1-6 次。

### OpenClaw 做法
- **Multi-agent routing**：deterministic routing, per-agent workspace, per-agent auth
- 多 agent 之间无自动协作循环
- 有 cross-agent memory search (`extraCollections`)

### MBclaw 当前状态
- ❌ 没有多模型/多 Key 支持
- ❌ 没有 agent 间协作

### 需要修改
1. **Key 管理**：`app/models/api_key.py` — 多个 key，每个标注能力和成本
2. **协作循环**：`app/services/collaboration_loop.py`
   ```
   Key1(executor) → 产出 → Key2(reviewer) → 评价+建议 → Key1 → 改进
   Loop 1-6 次或直到 reviewer 评分 >= threshold
   ```
3. **Review 维度**：代码质量、逻辑正确性、安全性、完整性
4. **配置**：每个项目可设置循环次数、reviewer 严格度

### 问题/风险
- 双 Key 成本翻倍
- Key1 和 Key2 如果都是同一个模型的 API，可能存在系统性偏见（同一个模型的盲点相同）
- Reviewer 可能过于严格导致无限循环

---

## 项目六：实时记忆预调用

### 需求
AI 思考时不只用固定记忆，根据用户当前对话内容，实时预调用项目二总结的知识，逐步深入调用更精准的信息。

### OpenClaw 做法
- **Memory search at session start**：每次 DM 会话开始时加载 MEMORY.md
- **Semantic memory_search / memory_get tools**：agent 可以主动搜索记忆
- **Hybrid search**：vector similarity + keyword matching
- **Extra collections**：cross-agent memory search

### MBclaw 当前状态
- ✅ MEMORY.md 已实现（Tier 1 持久记忆）
- ✅ Daily notes 已实现（Tier 2 今天+昨天自动加载）
- ✅ `/api/search` 全文搜索
- ❌ 搜索是被动的（agent 需要主动调用），不是"根据对话内容自动预调用"
- ❌ 没有向量语义搜索（当前是 SQLite LIKE）
- ❌ 没有分层递进搜索（粗略 → 精确）

### 需要修改
1. **预调用触发器**：在每轮对话前自动搜索相关记忆，注入到 agent 上下文
2. **分层搜索**：
   - L1: 关键词快速匹配（SQLite LIKE，<10ms）
   - L2: TF-IDF 相似度（当前 jieba，<100ms）
   - L3: 向量语义搜索（需要 embedding，<500ms via API）
3. **上下文注入**：搜索结果以 `[相关记忆]` 块形式注入 prompt，而非等待 agent 调用
4. **渐进加载**：先注入 L1+L2，如果需要更多细节再触发 L3

### 问题/风险
- 预调用增加延迟（每次对话前都要搜索）
- 注入过多记忆可能填满上下文
- embedding API 费用（如果用云端 embedding）

---

## 项目七：用户消息最高优先级

### 需求
跑任务时不能忽略用户最新消息。OpenClaw 默认把新消息排到最后，MBclaw 改成：最新用户消息 = 最新任务，不排队。同时不杀旧任务，放后台。

### OpenClaw 做法
- **Queue modes**：steer / followup / collect / interrupt
- **Per-session serialized lanes**：每个 session 一个串行队列
- 新消息进入队列，等待当前 turn 完成
- Interrupt mode 可以打断当前执行

### MBclaw 当前状态
- ❌ 没有任务队列管理
- ❌ 没有后台任务 + 前台任务切换

### 需要修改
1. **任务优先级队列**：`app/services/task_queue.py`
   ```
   [Active: Task A (running in foreground)]
   [Paused: Task B, Task C (background, state saved)]
   [Queued: user requests waiting]
   ```
2. **行为**：
   - 用户新消息 → 立即中断当前 turn（safe point）→ 保存 Task A 状态 → 开始新任务
   - Task A 放后台，状态持久化到 SQLite
   - 新任务完成后自动恢复 Task A（或等用户指令）
3. **安全中断点**：不是一个指令执行到一半就中断，而是在当前 tool call 完成后中断
4. **与 OpenClaw queue modes 对比**：
   - OpenClaw `steer`：新消息注入当前 session，不中断
   - OpenClaw `interrupt`：中断当前 turn，开始新消息
   - MBclaw 需要的：类似 `interrupt`，但被中断的任务保存状态到后台

### 问题/风险
- 中断正在执行的代码可能有副作用（文件写到一半）
- 状态保存和恢复复杂
- 与 OpenClaw 的 `interrupt` mode 几乎一样，是否直接复用？

---

## 项目八：中文优化

### 需求
全部报错命令翻译为中文并详解。默认配置界面中文翻译。能翻译的都翻译。

### OpenClaw 做法
- 英文为主，无官方中文支持
- 社区可能有 i18n 插件

### MBclaw 当前状态
- ✅ FastAPI 自动 OpenAPI 文档（英文）
- ❌ 没有中文错误消息
- ❌ 没有中文 UI/配置界面
- ✅ jieba 中文分词

### 需要修改
1. **错误消息中文化**：所有 HTTPException 返回中文 detail
2. **API 文档中文化**：FastAPI title/description/tags 改为中文
3. **配置界面**（如果有 Web UI）：全部中文
4. **安装/启动脚本**：中文注释

### 问题/风险
- 工作量不大但琐碎（每个 router、每个异常都要改）
- 中英文术语一致性（"session"翻译为"会话"还是"对话"？需要术语表）

---

## 项目九：删除无关检查

### 需求
检查 OpenClaw 各种启动检查、安全检查、配置检查，删除所有与 MBclaw 配置修改项目相关的检查。

### OpenClaw 做法
- **Config doctor**：`openclaw doctor --fix` 检测旧配置并迁移
- **Plugin gating**：`requires.bins` / `requires.env` / `requires.config` 自动禁用不可用插件
- **Sandboxing**：Docker/SSH/OpenShell 多种沙盒检查
- **Device identity**：WS 客户端设备身份验证
- **DM pairing**：未知发送者配对审批

### MBclaw 当前状态
- MBclaw-Lite 是白手起家写的，**不是 OpenClaw fork**
- ❌ 没有 OpenClaw 的检查系统（因为不是从 OpenClaw 改的）
- ✅ 代码干净，没有多余的检查

### 分析与修正

**重要澄清**：MBclaw-Lite 不是从 OpenClaw fork 出来的，而是独立开发的 FastAPI 应用。
不存在"删除 OpenClaw 检查"这个问题 — 因为从来没有 OpenClaw 的代码。

但如果用户计划**把 MBclaw 功能作为 OpenClaw 的插件/改装**安装进去，那么需要：

1. 识别 OpenClaw 中会阻止 MBclaw 功能的检查：
   - `config doctor` 可能标记未知配置为错误
   - 插件门控可能因缺少 bins/env 禁用 MBclaw 插件
   - 沙盒限制可能阻止 MBclaw 的文件写入
   - DM pairing 可能阻止 MBclaw 的自动消息
2. 对每项检查：评估是否可以安全跳过，还是需要适配

### 需要修改（如果是 Fork/改装方案）
- 见下文"关于 OpenClaw 改装方案"专题

### 问题/风险
- 删除安全检查可能导致安全隐患
- 需要逐项评估而不是全部删除

---

## 项目十：子对话协同改进

### 需求
子对话完成任务后自动反思，把结论发布到共享通道。其他子对话收到后直接用。启动新任务前先检查有没有别人做过了。遇到矛盾自动协商。整个过程自主，不需要主对话排练。

### OpenClaw 做法
- **Multi-agent with isolated workspaces**：每个 agent 独立的 session store
- **Cross-agent memory search**：`extraCollections` 可以搜索其他 agent 的记忆
- **No automatic collaboration loop**：agent 之间不自动通信

### MBclaw 当前状态
- ❌ 没有子对话/子 agent 概念
- ❌ 没有共享通道
- ❌ 没有自动协商

### 需要修改
1. **共享通道**：`app/models/shared_channel.py`
   - 每个 project 一个 shared_channel
   - agent 完成后自动 POST 反思结果
   - agent 启动前 GET 检查已有结果
2. **反思模板**（agent 完成后自动填写）：
   ```json
   {
     "agent_id": "sub-1",
     "task": "实现用户认证模块",
     "completed": true,
     "findings": ["JWT 比 session 更适合", "需要 refresh token"],
     "problems": ["CORS 配置有坑", "跨域 cookie 需要 SameSite=None"],
     "solutions": ["用 FastAPI middleware 统一处理 CORS"],
     "reusable": ["auth_middleware.py 可直接复用"],
     "conflicts": ["与 sub-2 的 API 路由冲突 → 需要协调"],
     "timestamp": "..."
   }
   ```
3. **去重检查**：`find_similar_tasks()` → 如果相似度 > 80%，直接用已有结果
4. **冲突协商**：两个 agent 任务冲突时，自动 LLM 调解（选一个方案或合并）

### 问题/风险
- 反思质量依赖 agent 的自我评估能力（可能不准确）
- 冲突协商需要额外 API 调用
- 共享通道可能变成瓶颈（多 agent 同时写）

---

## 项目十一：三层工具索引

### 需求
OpenClaw 经常忘记自己有什么工具。MBclaw 改进：
1. 添加工具时写 ~100 字简短介绍
2. 与项目二融合，空闲时/刚添加时将工具放入可能用得到的对话类型分类
3. 三层工具索引：轻量摘要 → 标签 → 完整描述
4. 向量语义搜索找工具（非关键词）
5. Token 预算控制：只注入相关工具

### OpenClaw 做法
- **Skills system**：SKILL.md + YAML frontmatter（`name`, `description`, `user-invocable`）
- **Skill snapshot**：session 开始时加载全部 skill 描述
- **问题**：skill 多了以后描述占大量 token，经常找不到合适的 skill

### MBclaw 当前状态
- ❌ 没有工具/技能系统
- ❌ 没有工具索引

### 需要修改
1. **工具注册表**：`app/models/tool.py`
   ```python
   class Tool:
       name: str
       summary: str          # ≤100 字
       tags: list[str]       # 自动+手动标签
       full_description: str # 完整文档
       embedding: list[float] # 文本向量（summary 的 embedding）
       usage_categories: list[str]  # 与项目二融合：适用场景
   ```
2. **三层索引**：
   - L1：100 字摘要（注入所有对话，Token 成本低）
   - L2：标签匹配（会话类型确定后，筛选相关工具）
   - L3：完整描述（确定使用某工具后，加载全部文档）
3. **向量搜索**：用 embedding API 将工具 summary 向量化，按语义相似度检索
4. **Token 预算**：每个对话类型的工具注入总量 < 可配置上限
5. **使用反馈**：记录工具使用频率/成功率，优先生成推荐

### 问题/风险
- embedding 存储需要向量数据库（或用 SQLite + json）
- 工具太多时，即使是 L1 摘要也可能超 token 预算
- 分类准确性依赖 LLM

---

## 项目十二：多模型智能调度

### 需求
用户添加多个不同模型的 Key 时，自动分析每个模型的特长（推理、图像、视频、网络搜索等），给模型打分标注，子任务自动分配最合适的模型 Key。

### OpenClaw 做法
- **Multi-provider plugins**：Anthropic / OpenAI / Ollama 等
- **Per-agent model config**：每个 agent 可以指定 model
- **Model selection**：agent 创建时静态指定，不动态调度

### MBclaw 当前状态
- ❌ 没有多模型管理
- ❌ 没有模型能力评估
- ❌ 没有成本感知调度

### 需要修改
1. **模型注册表**：`app/models/model_profile.py`
   ```python
   class ModelProfile:
       name: str               # "MiMo", "GPT-4o", "Claude 4"
       provider: str           # "mimo", "openai", "anthropic"
       api_key_ref: str        # 对应用户的哪个 key
       capabilities: dict      # {"reasoning": 0.9, "coding": 0.85, "vision": 0.0, ...}
       cost_per_1k_tokens: float
       max_tokens: int
       tool_compatibility: dict # {"browser": 0.8, "file_edit": 0.95, ...}
   ```
2. **能力评分**：自动 web search 获取模型特长，人工可覆盖
3. **联合优化调度器**：
   ```
   输入：任务类型 + 所需工具 + 预算上限
   输出：(model, tools) 最优组合
   算法：贪心 + 线性规划（简单版）
   ```
4. **成本感知**：
   - 简单任务 → 便宜模型
   - 复杂推理 → 最强模型
   - 用户可设置每任务/每日预算上限

### 问题/风险
- 能力评分依赖 web search 准确性
- 模型能力一直在更新（GPT-5 出来后所有评分要重算）
- 预算约束下的最优解可能不是用户想要的（太便宜导致质量差）

---

## 项目十三：MiMo Code 集成

### 需求
把 MiMo Code 集成到 MBclaw 配置页面，提供 MiMo Code 自带的限时免费一个月选项和与 OpenClaw 一样的 Key 配置选项。继续丢给 MiMo Code 跑，跑完检查是否出现了回滚我们改的地方。

### 分析

MiMo Code 是小米的 AI 编程助手。集成的意思是：
1. 在 MBclaw 配置页面增加 MiMo 的 API Key 配置
2. 支持 MiMo 的免费试用（一个月限制）
3. 用 MiMo Code 执行代码生成/修改任务
4. 任务完成后自动检查：MiMo 是否回滚了我们之前的修改

### 需要修改
1. **Provider 适配器**：`app/services/llm/mimo_adapter.py`
   - 封装 MiMo API 调用
   - 处理免费试用的 token 限制
2. **配置页面**：增加 MiMo Code 的 Key 输入 + 试用状态显示
3. **变更检测**：`app/services/change_detector.py`
   - 任务前后 git diff
   - 检测是否有意外的回滚（之前改的代码被抹掉）
   - 检测到回滚 → 告警 + 记录到 action_memory

### 问题/风险
- MiMo API 接口可能不标准（非 OpenAI 兼容格式）
- 免费试用可能限制 token 量或请求频率
- 回滚检测的假阳性（MiMo 可能合理重构了我们之前的代码）

---

# OpenClaw 改装方案

## 关键澄清

**MBclaw-Lite 是独立开发的 FastAPI 应用，不是 OpenClaw fork。**

用户说的"关于 OpenClaw 的改装方案"，应理解为：
- **方案 A**：把 MBclaw 功能做成 OpenClaw 的插件/扩展
- **方案 B**：以 OpenClaw 为底座，改装为 MBclaw（替换/删除/增加功能）
- **方案 C**：保持 MBclaw-Lite 独立，通过 API 与 OpenClaw 互通

## 方案对比

| 维度 | 方案 A（OpenClaw 插件） | 方案 B（OpenClaw 改装） | 方案 C（独立 + API 互通） |
|------|------------------------|------------------------|--------------------------|
| 开发难度 | 中（学习 OpenClaw SDK） | 高（需要深入 OpenClaw 内部） | 低（MBclaw 已完成） |
| 可维护性 | 低（跟随 OpenClaw 更新） | 最低（fork 维护成本极高） | 高（独立演进） |
| 功能完整度 | 受限（插件 API 限制） | 最高（无限制修改） | 中（API 调用有开销） |
| 用户消息优先级（项目七） | 插件层面难以实现 | 可以改 core | 独立控制 |
| 子对话协同（项目十） | 利用 OpenClaw multi-agent | 深度改造 | 自己实现 |
| 工具索引（项目十一） | 扩展现有 Skills 系统 | 替换 Skills 系统 | 独立建设 |
| 安全检查删除（项目九） | 插件可声明跳过检查 | 直接删除 | 不涉及 |
| 启动速度 | 依赖 OpenClaw 启动 | 同左 | 独立启动（快） |

## 推荐方案：C + A 混合

1. **MBclaw-Lite 保持独立**：作为记忆系统核心，独立运行，独立演化
2. **开发 OpenClaw Plugin**：把 MBclaw 的记忆能力通过 OpenClaw 插件 SDK 暴露给 OpenClaw agent
   - Memory search tool
   - Session save tool
   - Dreaming integration
3. **Webhook 互通**：OpenClaw session 完成 → webhook → MBclaw 存档
4. **不 Fork/不改装 OpenClaw**：避免维护地狱，保持两个项目的独立进化能力

## 如果坚持改装（方案 B），需要处理的检查

OpenClaw 的各种检查/门控如果阻碍 MBclaw 功能：

| 检查类型 | 是否需要删除 | 替代方案 |
|---------|-------------|---------|
| Config doctor | 修改（添加 MBclaw 配置 schema） | 不删除，扩展 |
| Plugin gating | 修改（声明 MBclaw requires） | 不删除，声明 |
| Sandbox 检查 | 修改（允许文件写入 data/） | 不删除，放宽 |
| DM pairing | 可选删除（MBclaw 场景不需要） | 替换为 API Key 验证 |
| Device identity | 可选删除 | 替换为 Token 验证 |
| Exec approvals | 修改（全自动模式下跳过） | 按模式动态切换 |
| Security audit | ⚠️ 不要删除 | 保留，增加 MBclaw 规则 |
| Session write lock | ⚠️ 不要删除 | 保留，防止数据损坏 |
