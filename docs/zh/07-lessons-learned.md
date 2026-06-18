# 07 — 经验总结与踩坑记录

## 技术决策

### 1. SQLite 足够好
对于单用户 MVP，SQLite 的 WAL 模式性能完全足够。不需要 PostgreSQL / MySQL。迁移成本极低（一个文件）。

### 2. FastAPI 选型正确
类型安全、自动 OpenAPI 文档、异步支持、生态丰富。未来接入 OpenHands / OpenClaw 只需 HTTP 调用。

### 3. proot 是 Android 上的唯一选择
尽管有性能损耗（~30%），但在小米设备上 CONFIG_USER_NS 大概率未开启，proot 是唯一可行的 Linux 环境方案。

### 4. DSP 固件不可绕过
小米设备的语音唤醒由 DSP 芯片独立处理，固件签名不可替换。MVP 阶段不应尝试替换，而应采用辅助唤醒（小爱 → MBclaw）方案。

### 5. Termux 是 Android Python 最成熟方案
不要自己交叉编译 CPython。Termux 维护了完整的 Android Python 生态。

### 6. API Only 是正确的方向
本地跑模型在手机上不现实。所有 LLM 通过 API 调用，本地只做存储和预处理。

### 7. OpenClaw 三层记忆是正确参考
MEMORY.md（持久）+ 每日笔记（半持久）+ Dreams（后台整合）的分层设计，比单一 SQLite 存储更符合 AI 记忆的认知模型。

## 架构经验

### 会话即边界
每次对话 = 一个 Session，总结 / 关键词 / 经验均以 Session 完成时为触发边界。这个设计保证了：
- 每次总结都有明确的上下文范围
- DNA 增量更新有清晰的触发时机
- 不会出现"该什么时候生成总结"的困惑

### JSON 字段 vs 关联表
Project DNA 的 goals / successful_approaches 等用 JSON Text 字段存储而非关联表，因为：
- 这些数据不需要单独查询
- 增量合并操作简单
- 减少了 JOIN 复杂度

### 文件锁保护写操作
memory_service 和 transcript_service 使用 fcntl 文件锁保护：
- 写操作使用 LOCK_EX（排他锁）
- 读操作使用 LOCK_SH（共享锁）
- 原子写入：先写 .tmp 文件，再 os.replace 到目标路径

### 搜索要跨所有维度
用户搜一个关键词，应该同时返回：
- 相关项目
- 相关会话
- 相关消息
- 相关总结
- 相关关键词

## 踩过的坑

1. **不要一开始就做多用户**：数据模型预留 user_id 即可，MVP 阶段先硬编码单用户
2. **不要做 Agent 自动化**：第一阶段只做记忆，不要实现复杂 Agent、多模型协作、自动执行
3. **系统镜像方案仅限于可重新打包刷机的场景**：Android 13+ erofs 不可写，运行时修改不现实
4. **语音唤醒的硬件限制比想象中大**：DSP 固件是真正的瓶颈
5. **文件写入要加锁**：即使是单用户 MVP，多进程并发也可能导致数据损坏
