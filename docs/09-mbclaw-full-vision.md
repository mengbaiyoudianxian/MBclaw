# 09 — MBclaw 完整愿景：13 项目改造方案（修订版）

> 基于 MBclaw 现有架构 + OpenClaw 参考 + 用户完整想法，逐项分析需要修改的地方。

## 先澄清几个关键点

### 1. MBclaw-Lite 不是 OpenClaw 的 Fork

MBclaw-Lite 是独立开发的 FastAPI 应用，不包含任何 OpenClaw 代码。所以：
- **项目九"删除 OpenClaw 检查"在 MBclaw-Lite 中不存在**：根本没有那些检查可以删。
- 只有当你决定把 MBclaw 作为 OpenClaw 的改装版时才需要处理检查问题。
- **推荐方案**：MBclaw 保持独立 + 开发 OpenClaw 插件把记忆能力暴露出去 = 方案 C+A（见文末）。

### 2. 项目七约等于 OpenClaw 的 interrupt mode

OpenClaw 已经有 `interrupt` queue mode（新消息打断当前 turn）。MBclaw 需要的是完全一样的东西，加一个"被打断的任务保存状态到后台"。不需要从零造。

### 3. 优先级不是平行的

项目二是所有记忆相关功能的基础。项目一、三、六都依赖项目二的分类体系。

---

## 项目一：详细日志备份

**核心需求**：记录改了什么代码 + 用户与 AI 对话 + AI 思考了什么，文件分片存储。

| 维度 | 现状 | 需要改 |
|------|------|--------|
| 对话内容 | ✅ JSONL transcript 已有 | — |
| 代码变更 | ❌ 不记录 | transcript 增加 `type: "code_change"` + `file_path` + `diff` |
| AI 思考 | ❌ 不记录 | assistant 消息增加 `thinking` 字段 |
| 文件分割 | ❌ 无 | 每文件最大 5MB，超出自动分片 |
| 文件大小膨胀 | — | thinking traces 可数 KB 每条，code diff 更大，需压缩策略 |

**改法**：升级 transcript JSON schema，增加 `type` 字段区分 message/code_change/thinking/decision。

---

## 项目二：空闲时自动整理对话

**核心需求**：
- 树状分类（粗略总结 → 详细内容，分支结构）
- 行不通方案的细节总结
- 关键词反向索引，AI 根据关键词搜索唤醒详细记忆
- 空闲时自动触发

| 维度 | 现状 | 需要改 |
|------|------|--------|
| 树状分类 | ❌ 无 | 按主题聚类对话，生成层级树 |
| 失败方案标记 | ❌ 无 | DNA 增加 `failed_approaches_detail`（结构化 JSON） |
| 关键词反向索引 | ❌ 无 | `keyword → [session_ids]` 映射表，O(1) 查找 |
| 定时/空闲触发 | ❌ 需外部 cron | `app/services/scheduler.py`，检测 API 空闲 |
| 分类引擎 | jieba 只能分词 | 需要 LLM 做语义分类（成本需控制） |

**这是所有项目中最基础的**。项目一、三、六、十一都依赖它的分类体系。

---

## 项目三：突破时备份

**核心需求**：项目有突破性进展时自动全量快照，防止后期修改破坏。

| 维度 | 现状 | 需要改 |
|------|------|--------|
| 版本快照 | ❌ 无 | DNA 的 successful_approaches 增加 → 自动 git commit + tag |
| 数据库备份 | ✅ SQLite 可热备份 | `VACUUM INTO` 或 `.backup` 到 `data/snapshots/` |
| 触发判断 | — | 关键词匹配"突破/bug fixed/解决了" + DNA 变更检测 |
| 假阳性 | — | 自动判断"突破"不准确，可能产生大量无用备份 |

---

## 项目四：全自动模式

**核心需求**：用户不确认时自己判断选最优方案，自动执行。用户没反应就自动研究其他选项，做几个成品让用户选。

| 维度 | 现状 | 需要改 |
|------|------|--------|
| 自动决策 | ❌ 无 | LLM 自评优劣 → 选最优 → 执行，记录理由到 action_memory |
| 多方案并行 | ❌ 无 | 多个 sub-agent 同时跑不同方案 |
| 安全边界 | — | 危险操作（rm -rf / 等）必须保留确认，不能全自动 |

---

## 项目五：双 Key 协作

**核心需求**：Key1 做产品，Key2 评价/找 bug/给方案，Key1 改。循环 1-6 次。

| 维度 | 现状 | 需要改 |
|------|------|--------|
| Key 管理 | ❌ 单 Key | `app/models/api_key.py` 多 Key + 能力标注 + 成本 |
| 协作循环 | ❌ 无 | Key1(executor) → Key2(reviewer) → 评价+建议 → Key1 改，循环 |
| 评审维度 | — | 代码质量、逻辑正确性、安全性、完整性 |
| 风险 | — | 同模型双 Key = 系统性偏见；循环次数需上限防无限 |

---

## 项目六：实时记忆预调用

**核心需求**：不只用固定记忆，根据当前对话实时预调用项目二的分类知识，逐步深入调用精准信息。

| 维度 | 现状 | 需要改 |
|------|------|--------|
| 搜索模式 | ❌ 被动（agent 主动调用） | 每轮对话前自动预搜索，结果注入上下文 |
| 分层搜索 | ❌ 只有 SQLite LIKE | L1 关键词(<10ms) → L2 TF-IDF(<100ms) → L3 向量语义(<500ms) |
| 上下文控制 | — | 注入量需控制，防止填满上下文 |
| embedding 成本 | — | L3 需要 embedding API，有费用 |

---

## 项目七：用户消息最高优先级

**核心需求**：最新用户消息 = 最新任务，不排队。旧任务不杀，放后台。

| 维度 | 现状 | 需要改 |
|------|------|--------|
| 任务队列 | ❌ 无 | `app/services/task_queue.py`，优先级队列 |
| 中断机制 | ❌ 无 | 新消息 → safe point 中断 → 保存旧任务状态 → 执行新任务 |
| OpenClaw 参考 | — | 本质就是 OpenClaw 的 `interrupt` queue mode + 状态持久化 |

---

## 项目八：中文优化

**核心需求**：全中文错误消息 + 中文配置界面 + 中文文档。

| 维度 | 现状 | 需要改 |
|------|------|--------|
| 错误消息 | ❌ 英文 | 所有 HTTPException 改为中文 |
| API 文档 | ❌ 英文 | FastAPI title/description/tags 中文化 |
| 术语统一 | — | 需要术语表：session→会话，project→项目，等 |

---

## 项目九：删除无关检查

**结论：此项目在当前 MBclaw-Lite 中不存在问题。**

MBclaw-Lite 是独立 FastAPI 应用，不含任何 OpenClaw 代码，无从删起。

- 只有当 Fork OpenClaw 改装时才有这个问题。
- 即使要改装，**Security audit 和 Session write lock 绝对不能删**。
- 推荐不 Fork，走独立 + 插件路线。

---

## 项目十：子对话协同改进

**核心需求**：子对话完成后自动反思，结论发布共享通道。启动前检查去重。矛盾自动协商。全程自主。

| 维度 | 现状 | 需要改 |
|------|------|--------|
| 共享通道 | ❌ 无 | `app/models/shared_channel.py`，每 project 一个 |
| 反思模板 | ❌ 无 | findings / problems / solutions / reusable / conflicts |
| 去重检查 | ❌ 无 | 任务相似度 > 80% → 复用已有结果 |
| 冲突协商 | ❌ 无 | LLM 调解：选最优或合并 |

---

## 项目十一：三层工具索引

**核心需求**：工具 ~100 字摘要 → 标签分类（与项目二融合）→ 详细描述。向量搜索。Token 预算控制。

| 维度 | 现状 | 需要改 |
|------|------|--------|
| 工具注册表 | ❌ 无 | `app/models/tool.py`，name + summary(≤100字) + tags + full_desc + embedding |
| 三层索引 | ❌ 无 | L1 摘要(全局注入) → L2 标签匹配 → L3 完整描述(按需加载) |
| 向量搜索 | ❌ 无 | embedding API 语义检索，非关键词匹配 |
| Token 预算 | ❌ 无 | 每个对话类型工具注入量 < 可配置上限 |

---

## 项目十二：多模型智能调度

**核心需求**：多 Key → 自动分析特长 → 打分标注 → 子任务自动分配最合适的模型。

| 维度 | 现状 | 需要改 |
|------|------|--------|
| 模型注册 | ❌ 无 | capabilities(0-1分值) + cost + tool_compatibility |
| 调度器 | ❌ 无 | 任务类型 + 所需工具 + 预算 → (model, tools) 联合优化 |
| 成本感知 | ❌ 无 | 简单任务→便宜模型；复杂推理→最强模型 |
| 能力评分 | — | web search + 人工覆盖；模型更新后需重评分 |

---

## 项目十三：MiMo Code 集成

**核心需求**：配置页面增加 MiMo Key 选项。用 MiMo 跑完自动检查是否回滚了之前的修改。

| 维度 | 现状 | 需要改 |
|------|------|--------|
| MiMo 适配 | ❌ 无 | `app/services/llm/mimo_adapter.py` |
| 变更检测 | ❌ 无 | 任务前后 git diff → 检测意外回滚 → 告警 |
| 风险 | — | MiMo API 可能非标准；免费试用有限制；回滚检测假阳性 |

---

# OpenClaw 改装方案：最终结论

## 三种可能路径

| 方案 | 描述 | 难度 | 推荐度 |
|------|------|------|--------|
| A | MBclaw 做成 OpenClaw 插件 | 中 | ⭐⭐⭐ |
| B | Fork OpenClaw 深度改装 | 极高 | ⭐ |
| C | MBclaw 独立运行 + API 互通 | 低 | ⭐⭐⭐⭐⭐ |

## 推荐：C + A

1. **MBclaw-Lite 保持独立** — 作为长期记忆系统核心
2. **OpenClaw 插件暴露记忆能力** — memory_search / session_save / dream 三个 tool
3. **Webhook 互通** — OpenClaw session 完成 → webhook → MBclaw 存档
4. **不 Fork OpenClaw** — 避免维护地狱

## 如果坚持方案 B（Fork 改装），项目九的处理

| OpenClaw 检查 | 处理 |
|--------------|------|
| Config doctor | 扩展，不删 |
| Plugin gating | 声明 requires，不删 |
| Sandbox 检查 | 放宽文件写入，不删 |
| DM pairing | 替换为 API Key 验证 |
| Device identity | 替换为 Token 验证 |
| Exec approvals | 按全自动模式动态切换 |
| Security audit | ⚠️ 绝对不能删 |
| Session write lock | ⚠️ 绝对不能删 |

## Phase 3 实施优先级

基于依赖关系排序：

```
项目二（分类体系）── 基础依赖
  ├── 项目一（详细日志）── 依赖分类体系归档
  ├── 项目三（突破备份）── 依赖分类体系判断"突破"
  ├── 项目六（实时预调用）── 依赖分类体系做分层搜索
  └── 项目十一（工具索引）── 依赖分类体系归类工具

项目七（消息优先级）── 独立，可并行
项目八（中文优化）── 独立，可并行
项目九（删检查）── 仅 Fork 方案时需要
项目十（子对话协同）── 依赖共享通道
项目十二（模型调度）── 依赖 Key 管理体系
项目四（全自动）── 依赖决策引擎
项目五（双 Key）── 依赖 Key 管理 + 协作循环
项目十三（MiMo）── 依赖 Provider 适配器
```
