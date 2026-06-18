# MBclaw — 长期记忆智能体系统

MBclaw 是一个让 AI 拥有长期记忆能力的系统。不仅记住聊天记录，还能记住项目目标、成功方案、失败方案、经验总结、关键词和后续计划。

## 仓库说明

本仓库存储 MBclaw 项目的全部设计思想、架构方案、技术决策和经验总结。
代码实现位于 [MBclaw-Lite](https://github.com/mengbaiyoudianxian/MBclaw-Lite)。

## 文档索引

| 文档 | 内容 |
|------|------|
| [01-项目愿景与目标](docs/01-vision.md) | 为什么要做 MBclaw，核心目标 |
| [02-MBclaw-Lite MVP架构](docs/02-mbclaw-lite-architecture.md) | FastAPI + SQLite 长期记忆核心设计 |
| [03-miclaw融合方案](docs/03-miclaw-integration.md) | MBclaw × miclaw Android 融合架构 |
| [04-系统镜像集成](docs/04-system-image-integration.md) | 系统级部署方案 |
| [05-MiMo纠错审查](docs/05-mimo-review.md) | MiMo 视角的架构纠错 |
| [06-三层客户端方案](docs/06-three-tier-client.md) | 非Root/Root/系统镜像三种模式 |
| [07-经验总结](docs/07-lessons-learned.md) | 技术决策与踩坑记录 |
