# 06 — 三层 Android 客户端方案

## 设计约束

- 不考虑本地和服务器跑模型，只考虑 API 接口
- 所有 LLM 通过 API 调用
- 本地 SQLite 用于数据存储
- 云端 MBclaw 服务器作为后端（API + 数据同步）

## 三种模式能力矩阵

| 能力 | 非 Root | Root (Magisk/KSU) | 系统镜像 |
|------|---------|-------------------|----------|
| MBclaw 对话 | ✅ | ✅ | ✅ |
| 本地 SQLite | ✅ | ✅ | ✅ |
| API 调用 LLM | ✅ | ✅ | ✅ |
| 语音唤醒 (辅助) | ✅ | ✅ | ✅ |
| 前台服务保活 | ✅ | ✅ | ✅ |
| 本地 Linux 沙盒 | ❌ | ✅ (proot) | ✅ (原生) |
| 开机自启动 | ❌ | ✅ | ✅ |
| 替换小爱语音 | ❌ | ❌ | ✅ |
| 系统级权限 | ❌ | ❌ | ✅ |
| 永不被杀 | ❌ | ❌ | ✅ |

## Mode 1 — 非 Root

标准 APK，类似普通应用：

```
MBclaw APK
├── WebView 对话界面
├── SQLite 本地存储
├── 前台服务保活
├── 辅助语音唤醒 (通过小爱打开)
└── HTTP → 云端 API (LLM)
```

**限制**：无 Linux 沙盒、无开机自启、无法替换小爱

## Mode 2 — Root (Magisk/KernelSU)

在非 Root 基础上增加：

```
+ Termux Python 环境
  ├── FastAPI MBclaw Core (127.0.0.1:8000)
  ├── proot Linux 沙盒
  └── 危险指令执行

+ Magisk 模块
  ├── 开机自启动脚本
  ├── 系统属性修改
  └── Magic Mount 系统文件

+ 云端同步
  └── SQLite ↔ 云服务器双向同步
```

## Mode 3 — 系统镜像

在 Root 基础上增加：

```
+ 系统服务 (priv-app)
  ├── init.rc 拉起守护进程
  ├── 永不被杀
  └── SELinux 自定义域

+ 语音框架替换
  ├── VoiceInteractionService 替换
  ├── DSP 唤醒词替换 (需重编译固件)
  └── 离线 ASR/TTS 模型

+ 原生 Linux 沙盒
  ├── unshare + chroot (零性能损耗)
  └── 完整 Ubuntu ARM64 环境

+ 系统接口全权限
```

## LLM API 调用链路

```
用户输入 → MBclaw APK → 本地 FastAPI (可选)
  → 云端 MBclaw Server → LLM API (MiMo/GPT/Claude)
  → 回复 → 本地 SQLite 存储 → TTS 语音输出
```

三种模式的区别仅在于：
- **非 Root**: 直接 HTTP 到云端
- **Root**: 可经过本地 FastAPI 预处理/缓存
- **系统镜像**: 本地 FastAPI 常驻 + 云端同步
