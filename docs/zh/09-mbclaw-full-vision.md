# 09 — MBclaw 完整愿景：13 项目改造方案

> 基于 MBclaw 现有架构 + OpenClaw 参考 + 用户完整想法
> 每项包含：需求分析、对比 OpenClaw 做法、MBclaw 差距、修改方案、潜在问题

---

## 项目一：详细日志备份

### 需求
完整记录每次对话的：改了什么代码、用户与 AI 对话内容、AI 思考了什么（thinking traces），全部详细备份到独立文件。每个文件大小合适，方便后续整理分类。

### OpenClaw 做法
- JSONL transcript：`<sessionId>.jsonl`，每行一条消息
- 文件级 fcntl 写锁

### MBclaw 当前状态
- ✅ `data/transcripts/{session_id}.jsonl` 已记录 role + content + timestamp
- ❌ 不记录"改了什么代码"（code diff/changes）
- ❌ 不记录"AI 思考了什么"（thinking traces）
- ❌ 文件大小不可控，没有自动分割

### 需要修改
1. **记录代码变更**：transcript 增加 `type: "code_change"`（file_path + diff）
2. **记录 thinking**：assistant 消息增加 `thinking` 字段
3. **文件分割**：每文件最大 5MB，超出自动分片
4. **Risk**：thinking traces 可能很大（OpenHands thinking 块可达数 KB），code diff 存储会让文件膨胀快

---

## 项目二：空闲时自动整理对话

### 需求
空闲时刻调用 API 自动分析所有对话，按主题类型分类成树状结构（粗略总结 → 详细内容），包含行不通方案的细节。给每个对话提取关键词，AI 需要时根据关键词搜索唤醒详细记忆。

### OpenClaw 做法
- Dreaming：后台整合，score → filter → promote
- Memory Wiki：结构化知识页面
- Memory Flush：上下文压缩前静默保存

### MBclaw 当前状态
- ✅ Dreaming 已实现
- ✅ Memory Flush 已实现
- ✅ 关键词提取（jieba + TF-IDF）
- ❌ 不做树状分类（粗略→详细的分支结构）
- ❌ 不做"行不通方案"专项标记
- ❌ 没有定时/空闲触发器

### 需要修改
1. **树状分类引擎**：按主题聚类对话，生成层级结构
2. **失败方案专项存储**：Project DNA 增加 `failed_approaches_detail`（结构化 JSON）
3. **空闲调度器**：`app/services/scheduler.py`
4. **关键词反向索引**：`keyword → [session_ids]` 映射表，O(1) 查找
5. **Risk**：树状分类需要 LLM 调用（本地 jieba 做不到语义分类），中文分类准确度依赖 LLM 质量

---

## 项目三：突破时备份

### 需求
项目有突破时，自动快照备份，防止后期修改破坏成果。

### MBclaw 当前状态
- ❌ 没有项目快照/版本管理
- ✅ SQLite 数据库（可用 `VACUUM INTO` 热备份）

### 需要修改
1. **Git 自动快照**：DNA 的 successful_approaches 增加 → 自动 `git commit + tag`
2. **数据库热备份**：`sqlite3 .backup`
3. **触发条件**：DNA 变更 / 用户标记 / 关键词匹配"突破/bug fixed/解决了"
4. **Risk**：自动判断"突破"不准确，可能产生大量无用备份

---

## 项目四：全自动模式

### 需求
用户要求全自动时，不需要用户确认就自己判断哪个方案更好，自动执行。用户没反应就自动研究其他选项，做几个成品让用户选。

### OpenClaw 做法
- Commitments（推断的短期后续行动）
- 没有"全自动无障碍模式"

### 需要修改
1. **自动决策引擎**：LLM 自评各方案优劣 → 选最优 → 执行
2. **多方案并行**：多个 sub-agent 同时跑不同方案
3. **安全边界**：危险操作（rm -rf /、修改系统配置）必须确认
4. **Risk**：安全性（与项目九配合）、成本（N 倍 API 调用）

---

## 项目五：双 Key 协作

### 需求
Key1 做产品，Key2 评价/找 bug/给改进方案，Key1 继续改。循环 1-6 次。

### OpenClaw 做法
- Multi-agent routing、per-agent auth
- 无自动协作循环

### 需要修改
1. **Key 管理**：`app/models/api_key.py` — 多个 key，标注能力和成本
2. **协作循环**：Key1(executor) → 产出 → Key2(reviewer) → 评价+建议 → Key1 改进
3. **Review 维度**：代码质量、逻辑正确性、安全性、完整性
4. **Risk**：成本翻倍，同一模型可能存在系统性偏见

---

## 项目六：实时记忆预调用

### 需求
AI 思考时根据用户当前对话内容，实时预调用项目二的总结知识，逐步深入调用更精准的信息。

### OpenClaw 做法
- Semantic memory_search / memory_get tools
- Hybrid search：vector similarity + keyword matching

### MBclaw 当前状态
- ✅ MEMORY.md + Daily notes + Dreaming 都已实现
- ❌ 搜索是被动的（agent 需要主动调用）
- ❌ 没有向量语义搜索
- ❌ 没有分层递进搜索

### 需要修改
1. **预调用触发器**：每轮对话前自动搜索相关记忆，注入上下文
2. **分层搜索**：L1 关键词匹配 → L2 TF-IDF → L3 向量语义搜索
3. **Risk**：预调用增加延迟，注入过多记忆可能填满上下文

---

## 项目七：用户消息最高优先级

### 需求
跑任务时不能忽略用户最新消息。最新用户消息 = 最新任务，不排队。不杀旧任务，放后台。

### OpenClaw 做法
- Queue modes：steer / followup / collect / interrupt
- Interrupt mode 可以打断当前执行

### 需要修改
1. **任务优先级队列**：`app/services/task_queue.py`
2. **行为**：新消息 → safe point 中断 → 保存旧任务状态 → 执行新任务
3. **Risk**：中断可能有副作用（文件写到一半），状态恢复复杂

---

## 项目八：中文优化

### 需求
全部报错命令翻译为中文并详解。配置界面中文翻译。

### 需要修改
1. 所有 HTTPException 返回中文 detail
2. API 文档中文化
3. **Risk**：琐碎但量大，需要术语表统一翻译

---

## 项目九：删除无关检查

### 重要澄清
**MBclaw-Lite 是独立开发的 FastAPI 应用，不是 OpenClaw fork。**
不存在"删除 OpenClaw 检查"的问题 — 因为从来没有 OpenClaw 的代码。

### 如果是 OpenClaw 改装方案
需要处理的检查见下文"OpenClaw 改装方案"专题。

---

## 项目十：子对话协同改进

### 需求
子对话完成任务后自动反思，结论发布到共享通道。其他子对话直接用。启动前检查去重。矛盾自动协商。全程自主。

### OpenClaw 做法
- Multi-agent with isolated workspaces
- Cross-agent memory search (`extraCollections`)
- 无自动协作

### 需要修改
1. **共享通道**：`app/models/shared_channel.py`
2. **反思模板**：findings / problems / solutions / reusable / conflicts
3. **去重检查**：相似度 > 80% → 直接用已有结果
4. **冲突协商**：LLM 调解
5. **Risk**：反思质量依赖 agent 自我评估能力，共享通道可能成为瓶颈

---

## 项目十一：三层工具索引

### 需求
OpenClaw 经常忘记自己有什么工具。MBclaw 改进：
1. 添加工具时写 ~100 字介绍
2. 与项目二融合，空闲时分类
3. 三层索引：轻量摘要 → 标签 → 完整描述
4. 向量语义搜索
5. Token 预算控制

### 需要修改
1. **工具注册表**：`app/models/tool.py`
2. **三层索引**：L1 摘要（注入所有对话）、L2 标签匹配、L3 完整描述
3. **向量搜索**：embedding API 语义检索
4. **Token 预算**：每个对话类型工具注入量 < 可配置上限
5. **Risk**：embedding 存储开销，分类准确性依赖 LLM

---

## 项目十二：多模型智能调度

### 需求
多模型 Key 时自动分析特长，打分标注，子任务自动分配最合适的模型。

### OpenClaw 做法
- Multi-provider plugins
- Per-agent model config（静态指定）
- 无动态调度

### 需要修改
1. **模型注册表**：capabilities（0-1 分值）+ cost + tool_compatibility
2. **联合优化调度器**：任务类型 + 所需工具 + 预算 → (model, tools) 最优组合
3. **成本感知**：简单任务 → 便宜模型，复杂推理 → 最强模型
4. **Risk**：能力评分依赖 web search 准确性，评分需要持续更新

---

## 项目十三：MiMo Code 集成

### 需求
配置页面增加 MiMo Code Key 选项（免费试用一个月 + 与 OpenClaw 一样的 Key 配置）。用 MiMo Code 跑完自动检查是否回滚了我们的修改。

### 需要修改
1. **Provider 适配器**：`app/services/llm/mimo_adapter.py`
2. **配置页面**：MiMo Key 输入 + 试用状态显示
3. **变更检测**：`app/services/change_detector.py` — 任务前后 git diff，检测回滚
4. **Risk**：MiMo API 可能不标准，免费试用有限制，回滚检测假阳性

---

# OpenClaw 改装方案

## 关键澄清

**MBclaw-Lite 是独立开发的 FastAPI 应用，不是 OpenClaw fork。**

三种方案：

| 方案 | 描述 | 开发难度 | 推荐度 |
|------|------|---------|--------|
| **A：OpenClaw 插件** | MBclaw 功能做成 OpenClaw 插件 | 中 | ⭐⭐⭐ |
| **B：OpenClaw 改装** | 以 OpenClaw 为底座深度改造 | 极高 | ⭐ |
| **C：独立 + API 互通** | MBclaw 独立运行，通过 API 与 OpenClaw 互通 | 低 | ⭐⭐⭐⭐⭐ |

### 推荐：C + A 混合

1. **MBclaw-Lite 保持独立**：作为记忆系统核心，独立运行和演化
2. **开发 OpenClaw Plugin**：把记忆能力通过插件 SDK 暴露给 OpenClaw agent
3. **Webhook 互通**：OpenClaw session 完成 → webhook → MBclaw 存档
4. **不 Fork OpenClaw**：避免维护地狱

### 如果坚持方案 B（改装），需要处理的检查

| 检查类型 | 处理方式 |
|---------|---------|
| Config doctor | 扩展（添加 MBclaw schema），不删除 |
| Plugin gating | 声明 MBclaw requires，不删除 |
| Sandbox 检查 | 放宽（允许文件写入），不删除 |
| DM pairing | 替换为 API Key 验证 |
| Device identity | 替换为 Token 验证 |
| Exec approvals | 按模式动态切换 |
| Security audit | ⚠️ 保留并增加 MBclaw 规则 |
| Session write lock | ⚠️ 保留 |
